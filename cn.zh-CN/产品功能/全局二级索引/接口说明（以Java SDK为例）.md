# 接口说明（以Java SDK为例） {#concept_r1l_4db_ffb .concept}

本文档以Java SDK为例，对createTable、scanFromIndex等接口进行详细说明。

-   创建主表的同时创建索引表。

    ```
    private static void createTable(SyncClient client) {
        TableMeta tableMeta = new TableMeta(TABLE_NAME);
        tableMeta.addPrimaryKeyColumn(new PrimaryKeySchema(PRIMARY_KEY_NAME_1, PrimaryKeyType.STRING)); // 为主表设置PK列
        tableMeta.addPrimaryKeyColumn(new PrimaryKeySchema(PRIMARY_KEY_NAME_2, PrimaryKeyType.INTEGER)); // 为主表设置PK列
        tableMeta.addDefinedColumn(new DefinedColumnSchema(DEFINED_COL_NAME_1, DefinedColumnType.STRING)); // 为主表设置预定义列
        tableMeta.addDefinedColumn(new DefinedColumnSchema(DEFINED_COL_NAME_2, DefinedColumnType.INTEGER)); // 为主表设置预定义列
        tableMeta.addDefinedColumn(new DefinedColumnSchema(DEFINED_COL_NAME_3, DefinedColumnType.INTEGER)); // 为主表设置预定义列
    
        int timeToLive = -1; // 数据过期时间设置为永不过期
        int maxVersions = 1; // 最大版本数为1（带索引表的主表只支持版本数为1）
    
        TableOptions tableOptions = new TableOptions(timeToLive, maxVersions);
    
        ArrayList<IndexMeta> indexMetas = new ArrayList<IndexMeta>();
        IndexMeta indexMeta = new IndexMeta(INDEX_NAME); // 新建索引表Meta
        indexMeta.addPrimaryKeyColumn(DEFINED_COL_NAME_1); // 指定主表的DEFINED_COL_NAME_1列作为索引表的PK
        indexMeta.addDefinedColumn(DEFINED_COL_NAME_2); // 指定主表的DEFINED_COL_NAME_2列作为索引表的属性列
        indexMetas.add(indexMeta); // 添加索引表到主表
    
        CreateTableRequest request = new CreateTableRequest(tableMeta, tableOptions, indexMetas); // 创建主表
    
        client.createTable(request);
    }
    ```

-   为已经存在的主表添加索引表。

    ```
    private static void createIndex(SyncClient client) {
        IndexMeta indexMeta = new IndexMeta(INDEX_NAME); // 新建索引Meta
        indexMeta.addPrimaryKeyColumn(DEFINED_COL_NAME_2); // 指定DEFINED_COL_NAME_2列为索引表的第一列PK
        indexMeta.addPrimaryKeyColumn(DEFINED_COL_NAME_1); // 指定DEFINED_COL_NAME_1列为索引表的第二列PK
        CreateIndexRequest request = new CreateIndexRequest(TABLE_NAME, indexMeta, false); // 将索引表添加到主表上
        client.createIndex(request); // 创建索引表
    }
    ```

    **说明：** 当前在添加索引表时，尚不支持索引表中包含主表中已经存在的数据。即新建索引表中将只包含主表从创建索引表开始时的增量数据。如果有对存量数据建索引的需求，请钉钉联系表格存储技术支持。

-   删除索引表。

    ```
    private static void deleteIndex(SyncClient client) {
        DeleteIndexRequest request = new DeleteIndexRequest(TABLE_NAME, INDEX_NAME); // 指定主表名称及索引表名称
        client.deleteIndex(request); // 删除索引表
    }
    ```

-   读取索引表中数据。

    需要返回的属性列在索引表中，您可以直接读取索引表：

    ```
    private static void scanFromIndex(SyncClient client) {
        RangeRowQueryCriteria rangeRowQueryCriteria = new RangeRowQueryCriteria(INDEX_NAME); // 设置索引表名
    
        // 设置起始主键
        PrimaryKeyBuilder startPrimaryKeyBuilder = PrimaryKeyBuilder.createPrimaryKeyBuilder();
        startPrimaryKeyBuilder.addPrimaryKeyColumn(DEFINED_COL_NAME_1, PrimaryKeyValue.INF_MIN); // 设置需要读取的索引列最小值
        startPrimaryKeyBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME_1, PrimaryKeyValue.INF_MIN); // 主表PK最小值
        startPrimaryKeyBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME_2, PrimaryKeyValue.INF_MIN); // 主表PK最小值
        rangeRowQueryCriteria.setInclusiveStartPrimaryKey(startPrimaryKeyBuilder.build());
    
        // 设置结束主键
        PrimaryKeyBuilder endPrimaryKeyBuilder = PrimaryKeyBuilder.createPrimaryKeyBuilder();
        endPrimaryKeyBuilder.addPrimaryKeyColumn(DEFINED_COL_NAME_1, PrimaryKeyValue.INF_MAX); // 设置需要读取的索引列最大值
        endPrimaryKeyBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME_1, PrimaryKeyValue.INF_MAX); // 主表PK最大值
        endPrimaryKeyBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME_2, PrimaryKeyValue.INF_MAX); // 主表PK最大值
        rangeRowQueryCriteria.setExclusiveEndPrimaryKey(endPrimaryKeyBuilder.build());
    
        rangeRowQueryCriteria.setMaxVersions(1);
    
        System.out.println("扫描索引表的结果为:");
        while (true) {
            GetRangeResponse getRangeResponse = client.getRange(new GetRangeRequest(rangeRowQueryCriteria));
            for (Row row : getRangeResponse.getRows()) {
                System.out.println(row);
            }
    
            // 若nextStartPrimaryKey不为null, 则继续读取.
            if (getRangeResponse.getNextStartPrimaryKey() != null) {
                rangeRowQueryCriteria.setInclusiveStartPrimaryKey(getRangeResponse.getNextStartPrimaryKey());
            } else {
                break;
            }
        }
    }
    ```

    需要返回的属性列不在索引表中，您需要反查主表：

    ```
    private static void scanFromIndex(SyncClient client) {
        RangeRowQueryCriteria rangeRowQueryCriteria = new RangeRowQueryCriteria(INDEX_NAME); // 设置索引表名
    
        // 设置起始主键
        PrimaryKeyBuilder startPrimaryKeyBuilder = PrimaryKeyBuilder.createPrimaryKeyBuilder();
        startPrimaryKeyBuilder.addPrimaryKeyColumn(DEFINED_COL_NAME_1, PrimaryKeyValue.INF_MIN); // 设置需要读取的索引列最小值
        startPrimaryKeyBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME_1, PrimaryKeyValue.INF_MIN); // 主表PK最小值
        startPrimaryKeyBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME_2, PrimaryKeyValue.INF_MIN); // 主表PK最小值
        rangeRowQueryCriteria.setInclusiveStartPrimaryKey(startPrimaryKeyBuilder.build());
    
        // 设置结束主键
        PrimaryKeyBuilder endPrimaryKeyBuilder = PrimaryKeyBuilder.createPrimaryKeyBuilder();
        endPrimaryKeyBuilder.addPrimaryKeyColumn(DEFINED_COL_NAME_1, PrimaryKeyValue.INF_MAX); // 设置需要读取的索引列最大值
        endPrimaryKeyBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME_1, PrimaryKeyValue.INF_MAX); // 主表PK最大值
        endPrimaryKeyBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME_2, PrimaryKeyValue.INF_MAX); // 主表PK最大值
        rangeRowQueryCriteria.setExclusiveEndPrimaryKey(endPrimaryKeyBuilder.build());
    
        rangeRowQueryCriteria.setMaxVersions(1);
    
        while (true) {
            GetRangeResponse getRangeResponse = client.getRange(new GetRangeRequest(rangeRowQueryCriteria));
            for (Row row : getRangeResponse.getRows()) {
                PrimaryKey curIndexPrimaryKey = row.getPrimaryKey();
                PrimaryKeyColumn pk1 = curIndexPrimaryKey.getPrimaryKeyColumn(PRIMARY_KEY_NAME1);
                PrimaryKeyColumn pk2 = curIndexPrimaryKey.getPrimaryKeyColumn(PRIMARY_KEY_NAME2);
                PrimaryKeyBuilder mainTablePKBuilder = PrimaryKeyBuilder.createPrimaryKeyBuilder();
                mainTablePKBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME1, pk1.getValue());
                mainTablePKBuilder.addPrimaryKeyColumn(PRIMARY_KEY_NAME2, ke2.getValue());
                PrimaryKey mainTablePK = mainTablePKBuilder.build(); // 根据索引表PK构造主表PK
    
                // 反查主表
                SingleRowQueryCriteria criteria = new SingleRowQueryCriteria(TABLE_NAME, mainTablePK);
                criteria.addColumnsToGet(DEFINED_COL_NAME3); // 读取主表的DEFINED_COL_NAME3列
                // 设置读取最新版本
                criteria.setMaxVersions(1);
                GetRowResponse getRowResponse = client.getRow(new GetRowRequest(criteria));
                Row mainTableRow = getRowResponse.getRow();
                System.out.println(row); 
            }
    
            // 若nextStartPrimaryKey不为null, 则继续读取.
            if (getRangeResponse.getNextStartPrimaryKey() != null) {
                rangeRowQueryCriteria.setInclusiveStartPrimaryKey(getRangeResponse.getNextStartPrimaryKey());
            } else {
                break;
            }
        }
    }
    ```


