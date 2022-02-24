# PROJECT #3 - QUERY EXECUTION

## OVERVIEW

+ 实现数据库的查询功能，即补充`executor`的核心代码`Init()`和`Next()`
  1. `Access Methods`：*seq_scan_executor*
  2. `Modifications`：*insert_executor*, *delete_executor*, *update_executor*
  3. `Miscellaneous`：*nested_loop_join_executor*

## SRC

- `src/include/execution/executors/seq_scan_executor.h`
- `src/include/execution/executors/index_scan_executor.h`
- `src/include/execution/executors/insert_executor.h`
- `src/include/execution/executors/update_executor.h`
- `src/include/execution/executors/delete_executor.h`
- `src/include/execution/executors/nested_loop_join_executor.h`
- `src/include/execution/executors/nested_index_join_executor.h`
- `src/include/execution/executors/aggregation_executor.h`
- `src/include/execution/executors/limit_executor.h`

## EXECUTOR

+ `SEQUENTIAL SCAN`：利用`iterator`实现遍历`table`的功能
+ `INSERT`
+ `UPDATE`
+ `DELETE`
+ `NESTED LOOP JOIN`

## IMPLEMENTATION

1. `SEQUENTIAL SCAN`

   ```c++
   /**
    * The SeqScanExecutor executor executes a sequential table scan.
    */
   class SeqScanExecutor : public AbstractExecutor {
    public:
     /**
      * Construct a new SeqScanExecutor instance.
      * @param exec_ctx The executor context
      * @param plan The sequential scan plan to be executed
      */
     SeqScanExecutor(ExecutorContext *exec_ctx, const SeqScanPlanNode *plan);
   
     /** Initialize the sequential scan */
     void Init() override;
   
     /**
      * Yield the next tuple from the sequential scan.
      * @param[out] tuple The next tuple produced by the scan
      * @param[out] rid The next tuple RID produced by the scan
      * @return `true` if a tuple was produced, `false` if there are no more tuples
      */
     bool Next(Tuple *tuple, RID *rid) override;
   
     /** @return The output schema for the sequential scan */
     const Schema *GetOutputSchema() override { return plan_->OutputSchema(); }
   
    private:
     /** The sequential scan plan node to be executed */
     const SeqScanPlanNode *plan_;
   
     TableInfo *tableMeta_;
     TableHeap *table_;
     TableIterator tableIte_;
     Schema schema_;
     std::list<std::pair<Tuple, RID>> values_;
   };
   
   SeqScanExecutor::SeqScanExecutor(ExecutorContext *exec_ctx, const SeqScanPlanNode *plan)
       : AbstractExecutor(exec_ctx), plan_(plan),
         tableMeta_(exec_ctx_->GetCatalog()->GetTable(plan_->GetTableOid())),
         table_(tableMeta_->table_.get()),
         tableIte_(TableIterator(table_->Begin(exec_ctx_->GetTransaction()))),
         schema_(*GetOutputSchema()) {}
   
   void SeqScanExecutor::Init() {
     auto predicate = plan_->GetPredicate();
   
     while(tableIte_ != table_->End()) {
       if(predicate) {
         bool isRight = predicate->Evaluate(&(*tableIte_), &schema_).GetAs<bool>();
   
         if(isRight) {
           values_.push_back(std::pair<Tuple, RID>(*tableIte_, tableIte_->GetRid()));
         }
       }
       else {
         values_.push_back(std::pair<Tuple, RID>(*tableIte_, tableIte_->GetRid()));
       }
       tableIte_++;
     }
   }
   
   bool SeqScanExecutor::Next(Tuple *tuple, RID *rid) {
     if(values_.empty()) {
       return false;
     }
   
     *tuple = values_.front().first;
     *rid = values_.front().second;
     values_.pop_front();
   
     return true;
   }
   
   ```

   + 需要注意，在`Init()`过程中遍历`table`，记录下每个`Tuple`存放在`values_`中

2. `INSERT`

   ```C++
   /**
    * InsertExecutor executes an insert on a table.
    *
    * Unlike UPDATE and DELETE, inserted values may either be
    * embedded in the plan itself or be pulled from a child executor.
    */
   class InsertExecutor : public AbstractExecutor {
    public:
     /**
      * Construct a new InsertExecutor instance.
      * @param exec_ctx The executor context
      * @param plan The insert plan to be executed
      * @param child_executor The child executor from which inserted tuples are pulled (may be `nullptr`)
      */
     InsertExecutor(ExecutorContext *exec_ctx, const InsertPlanNode *plan,
                    std::unique_ptr<AbstractExecutor> &&child_executor);
   
     ~InsertExecutor();
   
     /** Initialize the insert */
     void Init() override;
   
     /**
      * Yield the next tuple from the insert.
      * @param[out] tuple The next tuple produced by the insert
      * @param[out] rid The next tuple RID produced by the insert
      * @return `true` if a tuple was produced, `false` if there are no more tuples
      *
      * NOTE: InsertExecutor::Next() does not use the `tuple` out-parameter.
      * NOTE: InsertExecutor::Next() does not use the `rid` out-parameter.
      */
     bool Next([[maybe_unused]] Tuple *tuple, RID *rid) override;
   
     /** @return The output schema for the insert */
     const Schema *GetOutputSchema() override { return plan_->OutputSchema(); };
   
    private:
     /** The insert plan node to be executed*/
     const InsertPlanNode *plan_;
   
     std::unique_ptr<AbstractExecutor> childExec_;
   
     TableInfo *tableMeta_;
     TableHeap *table_;
     std::vector<IndexInfo*> indexs_;
     Schema *schema_;
     std::list<Tuple> values_;
   };
     
   InsertExecutor::InsertExecutor(ExecutorContext *exec_ctx, const InsertPlanNode *plan,
                                  std::unique_ptr<AbstractExecutor> &&child_executor)
                                  : AbstractExecutor(exec_ctx), plan_(plan),
                                  childExec_(std::move(child_executor)) {}
   
   InsertExecutor::~InsertExecutor() {
     delete schema_;
   }
   
   void InsertExecutor::Init() {
     tableMeta_ = exec_ctx_->GetCatalog()->GetTable(plan_->TableOid());
     table_ = tableMeta_->table_.get();
     indexs_ = exec_ctx_->GetCatalog()->GetTableIndexes(tableMeta_->name_);
   
     if(GetOutputSchema() == nullptr) {
       auto tableOid = plan_->TableOid();
       TableInfo *talbeInfo = exec_ctx_->GetCatalog()->GetTable(tableOid);
       schema_ = new Schema(talbeInfo->schema_);
     }
     else {
       schema_ = new Schema(*GetOutputSchema());
     }
   
     if(plan_->IsRawInsert()) {
       auto rawValues = plan_->RawValues();
       for(auto &v : rawValues) {
         values_.push_back(Tuple(v, schema_));
       }
     }
     else {
       auto context = childExec_->GetExecutorContext();
       auto engine = std::make_unique<ExecutionEngine>(context->GetBufferPoolManager(),
                                                       context->GetTransactionManager(), context->GetCatalog());
       auto plan = plan_->GetChildPlan();
   
       std::vector<Tuple> results;
       engine->Execute(plan, &results, context->GetTransaction(), context);
   
       for(auto &v : results) {
         values_.push_back(v);
       }
     }
   }
   
   bool InsertExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) {
     if(values_.empty()) {
       return false;
     }
   
     *tuple = values_.front();
     *rid = tuple->GetRid();
     values_.pop_front();
   
     table_->InsertTuple(*tuple, rid, exec_ctx_->GetTransaction());
   
     for(auto &v : indexs_) {
       v->index_->InsertEntry(*tuple, *rid, exec_ctx_->GetTransaction());
     }
   
     return true;
   }
   ```

   + 需要注意，当`child_executor`为空时，需要指定一个`plan`，否则无法准确定位每个字段内容

3. `UPDATE`

   ```c++
   /**
    * UpdateExecutor executes an update on a table.
    * Updated values are always pulled from a child.
    */
   class UpdateExecutor : public AbstractExecutor {
     friend class UpdatePlanNode;
   
    public:
     /**
      * Construct a new UpdateExecutor instance.
      * @param exec_ctx The executor context
      * @param plan The update plan to be executed
      * @param child_executor The child executor that feeds the update
      */
     UpdateExecutor(ExecutorContext *exec_ctx, const UpdatePlanNode *plan,
                    std::unique_ptr<AbstractExecutor> &&child_executor);
   
     ~UpdateExecutor();
   
     /** Initialize the update */
     void Init() override;
   
     /**
      * Yield the next tuple from the udpate.
      * @param[out] tuple The next tuple produced by the update
      * @param[out] rid The next tuple RID produced by the update
      * @return `true` if a tuple was produced, `false` if there are no more tuples
      *
      * NOTE: UpdateExecutor::Next() does not use the `tuple` out-parameter.
      * NOTE: UpdateExecutor::Next() does not use the `rid` out-parameter.
      */
     bool Next([[maybe_unused]] Tuple *tuple, RID *rid) override;
   
     /** @return The output schema for the update */
     const Schema *GetOutputSchema() override { return plan_->OutputSchema(); };
   
    private:
     /**
      * Given a tuple, creates a new, updated tuple
      * based on the `UpdateInfo` provided in the plan.
      * @param src_tuple The tuple to be updated
      */
     Tuple GenerateUpdatedTuple(const Tuple &src_tuple);
   
     /** The update plan node to be executed */
     const UpdatePlanNode *plan_;
     /** Metadata identifying the table that should be updated */
     const TableInfo *table_info_;
     /** The child executor to obtain value from */
     std::unique_ptr<AbstractExecutor> child_executor_;
   
     TableHeap *table_;
     TableIterator tableIte_;
     std::vector<IndexInfo*> indexs_;
     Schema *schema_;
     SeqScanPlanNode* childPlan_;
     std::list<std::pair<Tuple, RID>> values_;
   };
   
   UpdateExecutor::UpdateExecutor(ExecutorContext *exec_ctx, const UpdatePlanNode *plan,
                                  std::unique_ptr<AbstractExecutor> &&child_executor)
                                  : AbstractExecutor(exec_ctx), plan_(plan),
                                   table_info_(exec_ctx_->GetCatalog()->GetTable(plan_->TableOid())),
                                   child_executor_(std::move(child_executor)),
                                  table_(table_info_->table_.get()),
                                  tableIte_(TableIterator(table_->Begin(exec_ctx_->GetTransaction()))) {}
   
   UpdateExecutor::~UpdateExecutor() {
     delete schema_;
   }
   
   void UpdateExecutor::Init() {
     indexs_ = exec_ctx_->GetCatalog()->GetTableIndexes(table_info_->name_);
   
     if(GetOutputSchema() == nullptr) {
       auto tableOid = plan_->TableOid();
       TableInfo *talbeInfo = exec_ctx_->GetCatalog()->GetTable(tableOid);
       schema_ = new Schema(talbeInfo->schema_);
     }
     else {
       schema_ = new Schema(*GetOutputSchema());
     }
   
     childPlan_ = (SeqScanPlanNode *)plan_->GetChildPlan();
     auto predicate = childPlan_->GetPredicate();
   
     while(tableIte_ != table_->End()) {
       if(predicate) {
         bool isRight = predicate->Evaluate(&(*tableIte_), schema_).GetAs<bool>();
   
         if(isRight) {
           auto tuple = GenerateUpdatedTuple(*tableIte_);
           table_->UpdateTuple(tuple, tableIte_->GetRid(), exec_ctx_->GetTransaction());
         }
       }
       else {
         auto tuple = GenerateUpdatedTuple(*tableIte_);
         table_->UpdateTuple(tuple, tableIte_->GetRid(), exec_ctx_->GetTransaction());
       }
       tableIte_++;
     }
   }
   
   bool UpdateExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) { return false; }
   ```

   + 与`SEQUENTIAL SCAN`类似

4. `DELETE`

   ```c++
   /**
    * DeletedExecutor executes a delete on a table.
    * Deleted values are always pulled from a child.
    */
   class DeleteExecutor : public AbstractExecutor {
    public:
     /**
      * Construct a new DeleteExecutor instance.
      * @param exec_ctx The executor context
      * @param plan The delete plan to be executed
      * @param child_executor The child executor that feeds the delete
      */
     DeleteExecutor(ExecutorContext *exec_ctx, const DeletePlanNode *plan,
                    std::unique_ptr<AbstractExecutor> &&child_executor);
   
     ~DeleteExecutor();
   
     /** Initialize the delete */
     void Init() override;
   
     /**
      * Yield the next tuple from the delete.
      * @param[out] tuple The next tuple produced by the update
      * @param[out] rid The next tuple RID produced by the update
      * @return `false` unconditionally (throw to indicate failure)
      *
      * NOTE: DeleteExecutor::Next() does not use the `tuple` out-parameter.
      * NOTE: DeleteExecutor::Next() does not use the `rid` out-parameter.
      */
     bool Next([[maybe_unused]] Tuple *tuple, RID *rid) override;
   
     /** @return The output schema for the delete */
     const Schema *GetOutputSchema() override { return plan_->OutputSchema(); };
   
    private:
     /** The delete plan node to be executed */
     const DeletePlanNode *plan_;
     /** The child executor from which RIDs for deleted tuples are pulled */
     std::unique_ptr<AbstractExecutor> child_executor_;
   
     TableInfo *tableMeta_;
     TableHeap *table_;
     TableIterator tableIte_;
     std::vector<IndexInfo*> indexs_;
     Schema *schema_;
     SeqScanPlanNode* childPlan_;
     std::list<std::pair<Tuple, RID>> values_;
   };
   
   DeleteExecutor::DeleteExecutor(ExecutorContext *exec_ctx, const DeletePlanNode *plan,
                                  std::unique_ptr<AbstractExecutor> &&child_executor)
                  : AbstractExecutor(exec_ctx), plan_(plan), child_executor_(std::move(child_executor)),
                  tableMeta_(exec_ctx_->GetCatalog()->GetTable(plan_->TableOid())),
                  table_(tableMeta_->table_.get()),
                  tableIte_(TableIterator(table_->Begin(exec_ctx_->GetTransaction()))) {}
   
   DeleteExecutor::~DeleteExecutor() {
     delete schema_;
   }
   
   void DeleteExecutor::Init() {
     indexs_ = exec_ctx_->GetCatalog()->GetTableIndexes(tableMeta_->name_);
   
     if(GetOutputSchema() == nullptr) {
       auto tableOid = plan_->TableOid();
       TableInfo *talbeInfo = exec_ctx_->GetCatalog()->GetTable(tableOid);
       schema_ = new Schema(talbeInfo->schema_);
     }
     else {
       schema_ = new Schema(*GetOutputSchema());
     }
   
     childPlan_ = (SeqScanPlanNode *)plan_->GetChildPlan();
     auto predicate = childPlan_->GetPredicate();
   
     while(tableIte_ != table_->End()) {
       if(predicate) {
         bool isRight = predicate->Evaluate(&(*tableIte_), schema_).GetAs<bool>();
   
         if(isRight) {
           values_.push_back(std::pair<Tuple, RID>(*tableIte_, tableIte_->GetRid()));
         }
       }
       else {
         values_.push_back(std::pair<Tuple, RID>(*tableIte_, tableIte_->GetRid()));
       }
       tableIte_++;
     }
   }
   
   bool DeleteExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) {
     if(values_.empty()) {
       return false;
     }
   
     *tuple = values_.front().first;
     *rid = values_.front().second;
     values_.pop_front();
   
     /* 删除table的tuple和indexs中对应的rid */
     table_->MarkDelete(*rid, exec_ctx_->GetTransaction());
   
     return true;
   }
   ```

5. `NESTED LOOP JOIN`

   ```C++
   /**
    * NestedLoopJoinExecutor executes a nested-loop JOIN on two tables.
    */
   class NestedLoopJoinExecutor : public AbstractExecutor {
    public:
     /**
      * Construct a new NestedLoopJoinExecutor instance.
      * @param exec_ctx The executor context
      * @param plan The NestedLoop join plan to be executed
      * @param left_executor The child executor that produces tuple for the left side of join
      * @param right_executor The child executor that produces tuple for the right side of join
      */
     NestedLoopJoinExecutor(ExecutorContext *exec_ctx, const NestedLoopJoinPlanNode *plan,
                            std::unique_ptr<AbstractExecutor> &&left_executor,
                            std::unique_ptr<AbstractExecutor> &&right_executor);
   
     /** Initialize the join */
     void Init() override;
   
     /**
      * Yield the next tuple from the join.
      * @param[out] tuple The next tuple produced by the join
      * @param[out] rid The next tuple RID produced by the join
      * @return `true` if a tuple was produced, `false` if there are no more tuples
      */
     bool Next(Tuple *tuple, RID *rid) override;
   
     /** @return The output schema for the insert */
     const Schema *GetOutputSchema() override { return plan_->OutputSchema(); };
   
    private:
     /** The NestedLoopJoin plan node to be executed. */
     const NestedLoopJoinPlanNode *plan_;
   
     std::unique_ptr<AbstractExecutor> lhsExec_;
     std::unique_ptr<AbstractExecutor> rhsExec_;
   
     const SeqScanPlanNode *lhsPlan_;
     TableInfo *lhsTableMeta_;
     TableHeap *lhsTable_;
     TableIterator lhsTableIte_;
     const Schema lhsSchema_;
   
     const SeqScanPlanNode *rhsPlan_;
     TableInfo *rhsTableMeta_;
     TableHeap *rhsTable_;
     TableIterator rhsTableIte_;
     const Schema rhsSchema_;
   
     std::list<Tuple> values_;
   };
   
   NestedLoopJoinExecutor::NestedLoopJoinExecutor(ExecutorContext *exec_ctx, const NestedLoopJoinPlanNode *plan,
                          std::unique_ptr<AbstractExecutor> &&left_executor, std::unique_ptr<AbstractExecutor> &&right_executor)
              : AbstractExecutor(exec_ctx), plan_(plan),
              lhsExec_(std::move(left_executor)), rhsExec_(std::move(right_executor)),
              lhsPlan_((SeqScanPlanNode*)plan_->GetLeftPlan()),
              lhsTableMeta_(exec_ctx_->GetCatalog()->GetTable(lhsPlan_->GetTableOid())),
              lhsTable_(lhsTableMeta_->table_.get()),
              lhsTableIte_(lhsTable_->Begin(exec_ctx_->GetTransaction())), lhsSchema_(*lhsPlan_->OutputSchema()),
              rhsPlan_((SeqScanPlanNode*)plan_->GetRightPlan()),
              rhsTableMeta_(exec_ctx_->GetCatalog()->GetTable(rhsPlan_->GetTableOid())),
              rhsTable_(rhsTableMeta_->table_.get()),
              rhsTableIte_(rhsTable_->Begin(exec_ctx_->GetTransaction())), rhsSchema_(*rhsPlan_->OutputSchema()) {}
   
   void NestedLoopJoinExecutor::Init() {
     auto predicate = plan_->Predicate();
   
     while(lhsTableIte_ != lhsTable_->End()) {
       while(rhsTableIte_ != rhsTable_->End()) {
         if(predicate) {
           bool isRight = predicate->EvaluateJoin(&(*lhsTableIte_), &lhsSchema_, &(*rhsTableIte_), &rhsSchema_).GetAs<bool>();
   
           if(isRight) {
             std::vector<Value> results;
             for(uint32_t i=0; i<lhsSchema_.GetColumnCount(); i++) {
               results.push_back(lhsTableIte_->GetValue(&lhsSchema_, i));
             }
             for(uint32_t i=0; i<rhsSchema_.GetColumnCount(); i++) {
               results.push_back(rhsTableIte_->GetValue(&rhsSchema_, i));
             }
             auto tmpTuple = Tuple(results, GetOutputSchema());
             values_.push_back(tmpTuple);
           }
         }
         else {
           std::vector<Value> results;
           for(uint32_t i=0; i<lhsSchema_.GetColumnCount(); i++) {
             results.push_back(lhsTableIte_->GetValue(&lhsSchema_, i));
           }
           for(uint32_t i=0; i<rhsSchema_.GetColumnCount(); i++) {
             results.push_back(rhsTableIte_->GetValue(&rhsSchema_, i));
           }
           auto tmpTuple = Tuple(results, GetOutputSchema());
           values_.push_back(tmpTuple);
         }
         rhsTableIte_++;
       }
       lhsTableIte_++;
       rhsTableIte_ = rhsTable_->Begin(exec_ctx_->GetTransaction());
     }
   }
   
   bool NestedLoopJoinExecutor::Next(Tuple *tuple, RID *rid) {
     if(values_.empty()) {
       return false;
     }
   
     *tuple = values_.front();
     values_.pop_front();
   
     return true;
   }
   ```

   + 需要注意，求笛卡尔积是两张`table`的事
