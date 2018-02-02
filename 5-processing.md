5  **processing模块**

5.1  **Global Dictionary**

全局字典编码（目前为表级字典编码）使用统一的整数来代理真实数据，可以起到数据压缩作用，同时基于字典编码分组聚合的计算也更高效。全局字典编码生成过程如图所示，CarbonBlockDistinctValuesCombinRDD计算每个split内字典列的distinct value列表， 然后按照ColumnPartitioner做shuffle分区，每列一个分区。CarbonGlobalDictionaryGenerateRDD每个task处理一个字典列，生成字典文件（或者追加新字典值），刷新sortindex，更新dictmeta。在一次加载中，新生成的字典值和真实数据的顺序性是一致的，而在多次加载之间，该顺序性无法保持一致，建议仅对那些值不经常变化的列作字典编码。

<img src="media/5-1_1.png" width = "50%" alt="5-1_1" />

5.2  **DataLoading（数据加载）**

CSV文件数据加载主流程，如下图所示。首先，CarbonLoaderUtil.nodeBlockMapping将数据块按节点分区。NewCarbonDataLoadRDD采用该节点分区，每个节点启动一个task来处理数据块的加载。（**BlockAssignment**）DataLoadExecutor.execute执行数据加载，主流程包括图中所示的4个步骤。

<img src="media/5-2_1.png" width = "50%" alt="5-2_1" />

Step1:InputProcessorStepImpl使用CSVInputFormat.CSVRecordReader读取并解析CSV文件（**Read**）。

Step2:DataConverterProcessStepImpl用来转换数据（**Convert**）。FieldConverter主要有以下实现类，包括字典列，非字典列，直接字典，复杂类型列和度量列转换。
![5-2_2](media/5-2_2.png)
Step3: SortProcessorStepImpl将数据按照sort_columns进行sort，默认的Sorter实现类是(unsafe)ParallelReadMergeSorterImpl, Sorter主流程如右图所示。

SortDataRows.addRowBatch方法缓存数据，当数据记录数达到sort buffer size（默认100000，由carbon.sort.size设置）, 调用DataRorterAndWriter 排序（**TempSort**）并生成sorttemp file到local disk；当sorttemp file数量达到配置的阈值（默认20，由carbon.sort.intermediate.files.limit设置）调用SortIntermediateFileMerger.startMerge将这些tmpfile归并排序（**MergeSort**）生成big sorttemp file。 在Step1和Step2的输入数据都完成排序并生成文件（一些big tmpfile
和不到20个的tmpfile）到tmp目录后，SingleThreadFinalSortFilesMerger.startFinalMerge启动finalmerge，流式归并排序所有的tmpfile（**FinalSort**），目的是使本节点本次loading的数据有序，并为后续Step4提供数据的流式输入（FinalSort阶段边sort边为step4提供输入，数据不落地）。

<img src="media/5-2_3.png" width = "60%" alt="5-2_3" />

Step4:DataWriterProcessorStepImpl用于生成carbondata和carbonindex文件。主流程如下图所示。MultiDimKeyVarLengthGenerator.generateKey为每一行的字典编码dimesion生成MDK。CarbonFactDataHandlerColumnar.addDataToStore缓存MDK编码等数据，记录数达到blockletsize大小后，调用Producer生成Blocklet对象(代码中为NodeHolder)。（**Produce**）

BlockIndexerStorageForInt/Short处理blocklet内dimension列数据的排序(Array.sort)、生成RowId index(compressMyOwnWay),采用RLE压缩(compressDataMyOwnWay)

HeavyCompressedDoubleArrayDataInMemoryStore处理bloclet内meause类数据的压缩(使用snappy)。

AbstractFactDataWriter.createNewFileIfReachThreshold将已有的一个blocklet（也成为TablePage）数据写入本地数据文件（**Consume**）。如果blocklet累计大小已经达到了table\_blocksize（即建表时指定的table_blocksize，默认为1024MB）大小，新建一个carbondata文件来写入数据。

在carbondata file的blocklet写入结束后，调用writeBlockletInfoToFile完成footer部分写入。在本节点task结束后，调用writeIndexFile生成carbonindex文件。

最终生成的carbondata文件名类似于：part-0-0-batchno0-0-1517466908781.carbondata，文件名中的不同部分的含义为“part-fileNo-taskNo-batchno0-bucketNo-timestamp.carbondata”。其中taskNo是task号，一般是数据加载时的节点号；fileNo是当次加载时在该节点生成文件号；batchno0当前是固定值，暂时没有用；bucketNo根据不同的情况，可能是分桶号、分区号或其他含义。

最终生成的carbonindex文件文件名类似于：0-batchno0-0-1517466908781.carbonindex，文件名中不同部分的含义为“taskNo-batchNo0-bucketNo-timestamp.carbonindex”，各部分含义和前面carbondata中的一致。从这里可以看出，每次加载时，对一个节点只会生成一个carbonindex文件，这是对该节点生成的所有carbondata文件的索引，并且该文件比较小。如果集群规模较大，数据加载的节点数较多，则会造成单次加载生成的carbonindex文件较多，会影响后续查询的性能。在carbondata-1.3.0之后，carbondata在数据加载完成后，悔对carbonindex文件进行合并操作，生成的文件名后缀为“carbonindexmerge”。

<img src="media/5-2_4.png" width = "60%" alt="5-2_4" />

总结上述数据加载流程：

1. Carbondata首先将数据分配到各节点（**BlockAssignment**）；
2. 各节点分别处理自己的输入数据。首先解析CSV数据（**Read**）并分成batch，以流的方式提供输入；
3. Carbondata对每个batch的数据列进行转换（**Convert**）；
4. Carbondata对每个batch的数据进行排序（**TempSort**）并生成SortTempFile。每个SortTempFile内部是有序的，而之间是无序的；
5. Carbondata对SortTempFile进行小范围合并（**MergeSort**）并生成新的SortTempFile。该动作会发生多次，以小文件的个数为触发条件；
6. 最终轮排序（**FinalSort**）的输入是多个SortTempFile。Carbondata使用堆排序的方式，每个SortTempFile是堆中的一个节点。Carbondata不断从堆中取走最小的记录，以此达到全排序的目的。
7. Carbondata每取到32000行时，会生成（**Produce**）一个TablePage并放到环形队列中；
8. Carbondata依次从环形队列中取走TablePage，写入（**Consume**）Carbondata文件并最终生成carbonindex文件。



5.3  **Compression Encoding**

1. Snappy Compressor
2. Number Compressor
3. Delta Compressor
4. Dictionary Encoding
5. Direct-Ditionary Encoding
6. RLE(Running Length Encoding)
7. Adaptive * Encoding
8. Adaptive Delta * Encoding
