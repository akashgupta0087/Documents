# Sequence Run

- [Sequence Run](#sequence-run)
  - [Get Core Universe](#get-core-universe)
    - [Getting Universe from Given Table](#getting-universe-from-given-table)
    - [Getting Universe from Given Index](#getting-universe-from-given-index)
    - [Getting Universe from Entity Ids](#getting-universe-from-entity-ids)
  - [Get Derived Universe](#get-derived-universe)
    - [Getting Universe from ESG\_Services](#getting-universe-from-esg_services)
    - [Getting Universe from E3](#getting-universe-from-e3)
    - [Filtering Entities based on NOTs](#filtering-entities-based-on-nots)
  - [Forming Universe](#forming-universe)
  - [Load Cache](#load-cache)
    - [Load Entities Cache](#load-entities-cache)
    - [Load Sequences Cache](#load-sequences-cache)
    - [Load Factor Values Cache](#load-factor-values-cache)
  - [Actual Process Call](#actual-process-call)
    - [Process method of SCalculationData](#process-method-of-scalculationdata)
      - [Derived Universe Process](#derived-universe-process)
      - [Core Universe Process](#core-universe-process)
      - [Misc.](#misc)
  - [Excel Creation](#excel-creation)


## Get Core Universe
- We get the data for this universe from S_Universes Table. 
- This method `TreeSet<Integer> getUniverse_entityIDs(int sequenceID)` of class `com.iss.esg.sequence.SDataAccess` is called.
    Inside that we call the `SUniverseData getUniverseData_sequenceID(int sequenceID)` of the same class, which runs a SP : **`ssp_seq_getUniverseData2 `** and gives **SUniverseData** object.

    ```
    select u.* 
	from S_Sequences s 
	inner join S_Universes u on u.UniverseID = s.UniverseID
	where s.SequenceID = @_SequenceID
    ```
- Type of Universe is checked, whether it is T(Table), I(Index) or E(Entity).
    + For type T, universe data is fetched from the given table.
    ![universe_type_T]
    + For type I, universe data is fetched from the given index via Omega API
    ![universe_type_I]
    + For type E, we get the universe data which is list of entity Ids
    ![universe_type_E]

### Getting Universe from Given Table

- `TreeSet<Integer> getUniverse_entityIDs_T(SUniverseData ud)` method of class `SDataAccess` is called which calls SP : **`ssp_seq_getUniverse_entityIDs_T`**

    ```
    'select distinct t.EntityID from trove..' + @_TableName + ' t where t.EndDate = ''3999-12-31'' and t.EndDateType = ''O'' order by t.EntityID;';
        
    ```

### Getting Universe from Given Index

- `TreeSet<Integer> getUniverse_entityIDs_I(SUniverseData ud)` method of class SDataAccess is called, which internally calls `HashSet<Integer> getEntityIDs_indexID` method of `com.iss.esg.omega.ODataAccess` class, where an API call is to omega is made to fetch the data.
![omega_indexes_api]

- After parsing the response, we get the list of entity Ids

### Getting Universe from Entity Ids

- This we get directly from **SUniverseData** object which we got from S_Universes Table

## Get Derived Universe

- Here, we get the universe from ESG_Services Factor values and E3 Factor values
- `TreeSet<Integer> getUniverse_entityIDs_D(int sequenceID)` method of SDataAccess class is called.

### Getting Universe from ESG_Services

- We get the list(TreeSet) of factor values from `TreeSet<String> getFactors_ESG_D(int sequenceID)` of `com.iss.esg.sequence.SDataAccess` class, which internally calls `TreeSet<String> getFactors_ESG(int sequenceID, String storedProcedure)` of the same class, which calls the SP : **`ssp_seq_getFactors_ESG_D`**


    ```

    @_SequenceID = 27
    select distinct r.FactorName
        from S_SequenceXref x 
        inner join S_Calculations c on c.CalculationID = x.CalculationID and c.UniverseType = 'D'
        inner join S_FormulaXref fx on fx.FormulaID = x.FormulaID
        inner join S_Factors r on r.FactorID = fx.FactorID and r.FactorGroupType = 'ESG'
        where x.SequenceID = @_SequenceID
        order by r.FactorName
    ```
    ![esg_derived_factor_names]

- Now, `HashMap<Integer, HashMap<String, String>> getFactorValues_S(TreeSet<String> factorNames)` method of `com.iss.esg.service.SDataAccess` class is called and the factor names from the above query are supplied to it. Inside it `TreeMap<Integer, TreeSet<String>> getFactorGroups(TreeSet<String> factorNames_ts)` method of the same class is called, which calls SP : **`fgsp_fg_getFactorGroupsForFactors`**

    ```

    select distinct x.FactorGroupID, x.FactorName from (select fm.FactorGroupID, fm.FactorName, row_number() over (partition by fm.FactorName order by fm.FactorGroupID desc) as fm_version from FG_Factors fm where fm.FactorName in (' + @_FACTOR_NAMES + ')) x where x.fm_version = 1 order by x.FactorGroupID, x.FactorName
    ```
    and it gives a TreeMap of factorGroupId, TreeSet of FactorNames *<109, [CompNegAffectBioSensAreas, FossilFuelInvolvementPAI, InvolvInContrWeapons, LackProcessesUNGCOECDGuidelines, UNGCOECDGuidelinesViolation]>*

- After that `HashMap<Integer, HashMap<String, String>> getFactorValues(TreeMap<Integer, TreeSet<String>> factorGroups)` method of `com.iss.esg.service.SDataAccess` class is called, which internally calls `HashMap<Integer, HashMap<String, String>> getFactorValues(TreeMap<Integer, TreeSet<String>> factorGroups, int entityID, java.util.Date asOfDate)`, where the arguments supplied are factorGroups map, enitityID as 0 and asOfDate as null.
- Now , the above method makes call to `HashMap<Integer, HashMap<String, String>> getFactorValues(int factorGroupID, int entityID, java.util.Date asOfDate, String factorNames)` method of the same class.
  + TreeMap of <FactorGroupId, FactorNames> is iterated and for every factorGroup and API call is made.
  + API call is made to *http://vpna-dev-tmc081.ad-dev.issgovernance.com:8205/ESGServices/api/factor-groups/109/entities?factornames=CompNegAffectBioSensAreas|FossilFuelInvolvementPAI|InvolvInContrWeapons|LackProcessesUNGCOECDGuidelines|UNGCOECDGuidelinesViolation*
  ![factorgroups_entities_api]
    - After parsing the response, we get Hashmap of `<EntityID, <FactorName, FactorValue>>`
  + So for all the factorGroups we will end up getting the HashMap of `<EntityID, <FactorName, FactorValue>>`


### Getting Universe from E3

- We get the TreeMap from `TreeMap<Integer, TreeSet<String>> getFactors_E3_D(int sequenceID)` method of `com.iss.esg.sequence.SDataAccess` class which internally calls `TreeMap<Integer, TreeSet<String>> getFactors_E3(int sequenceID, String storedProcedure)` of the same class, which calls SP : **`ssp_seq_getFactors_E3_D`**

    ```

    @_SequenceID = 26
    select distinct r.FactorGroupID, r.FactorName
        from S_SequenceXref x 
        inner join S_Calculations c on c.CalculationID = x.CalculationID and c.UniverseType = 'D'
        inner join S_FormulaXref fx on fx.FormulaID = x.FormulaID
        inner join S_Factors r on r.FactorID = fx.FactorID and r.FactorGroupType = 'E3' and r.FactorGroupID != 0 and r.CaseQueryID = 0
        where x.SequenceID = @_SequenceID
        order by r.FactorGroupID, r.FactorName
    ```
    ![e3_derived_factor_names]

    It gives TreeMap of FactorGroupId, TreeSet of FactorNames
    For e.g. : 
    *<32,['CivFADistMaxRev', 'CivFAProdServMaxRev', 'CivFARevShareMax', 'MilitaryEqmtDistMaxRev', 'MilitaryEqmtProdServMaxRev', 'MilitaryEqmtRevShareMax']
    55, ['TobaccoDistMaxRev', 'TobaccoProdMaxRev', 'TobaccoRevShareMax', 'TobaccoServMaxRev']>*
- Now the above TreeMap is supplied as an input to `HashMap<Integer, HashMap<String, String>> getFactorValues_S(TreeMap<Integer, TreeSet<String>> factorGroups)` method of `com.iss.esg.ethix.EDataAccess` class, which internally calls `HashMap<Integer, HashMap<String, String>> getFactorValues(TreeMap<Integer, TreeSet<String>> factorGroups)` method of the same class
    + TreeMap of FactorGroupId, FactorNames is iterated on FactorGroupId, and for each FactorGroupId `HashMap<Integer, HashMap<String, String>> getFactorValues(int factorGroupID, String factorNames)` method of EDataAccess is called, which internally calls `HashMap<Integer, HashMap<String, String>> getFactorValues(int factorGroupID, java.util.Date asOfDate, String factorNames)` of the same class and the arguments supplied to it are the factorGroupId as 32, asOfDate as null and the factorNames as a String separated with '|'
    + API call is made to 
    + http://ethix-tools-uat.issapps.com/ethixdataservice/DataDeskFactorGroup/81/data?factors=ClimateNZTarget2050Scope3%7CClimateNZTargetInter95Pct%7CClimateNZTargetInterScope3
    ![e3_entityId_factornames_factorvalues]
        - After parsing the response, we get Hashmap of `<EntityID, <FactorName, FactorValue>>`
    + So for all the factorGroups we will end up getting the HashMap of `<EntityID, <FactorName, FactorValue>>`

### Filtering Entities based on NOTs

- Now both the Hashmaps(`<EntityID, <FactorName, FactorValue>>`) from ESG and E3 are checked for NOTs.
    + Take the entityId, if any one of its factors does not have **'NotCollected', 'NotApplicable', 'NotMeaningful', 'NotDisclosed',
    'NoInformation'** value in it
    + For e.g. If an entitiyId contains 10 factors and only 1 factor does not have NOTs in it, then add that EntityId in the Universe

## Forming Universe

- Now all the entities from Core and Derived universe are clubbed into one single Universe
![sdataaccess_getFactorValues]

## Load Cache

- `boolean LOAD_CACHE(int sequenceID, TreeSet<Integer> entityIDs, TreeSet<Integer> entityIDs_C)` method of `com.iss.esg.sequence.SDataCache` class is called which loads 3 types of cache:
  1. Entities Cache
  2. Sequences Cache
  3. FactorValues Cache

### Load Entities Cache

- `boolean LOAD_entities_CACHE(TreeSet<Integer> entityIDs, TreeSet<Integer> entityIDs_C)` method of `SDataCache` class is called where TreeSet of all the entityIds(Core + Derived) and Core EntityIds are passed as arguments
- Now inside the above method, `TreeMap<Integer, SEntityData> getEntities_optimizer(TreeSet<Integer> entityIDs, TreeSet<Integer> entityIDs_C)` is called of `com.iss.esg.SUtil` class, which iterates over all the entityIds and make an object of SEntityData for each entityId and set its core flag to true, if it belongs to **Core EntityId**, it returns a TreeMap of <EntityId, SEntityData>
- Now `getFactorValues_S(TreeMap<Integer, com.iss.esg.sequence.SEntityData> entities)` method of `com.iss.esg.omega.ODataAccess` class is called and the TreeMap obtained from the previous method is supplied to it as an input
    + Now, inside the above method, `HashMap<Integer, HashMap<String, String>> getFactorValues_coreDetails(HashSet<Integer> entityIDs)` method is called of the same class, where the entityIDs are divided into batches of 1000.
    + For every 1000 entityIds `HashMap<Integer, HashMap<String, String>> getCoreDetails(TreeSet<Integer> entityIDs)` method of the same class is called which makes an API call to Omega
        - API call is made to *http://vpna-int-msc001.ad-dev.issgovernance.com:2222/OmegaServices/entities/coreDetails*
        - Request body is passed as OBodyData, which consists of *`ArrayList<String> entityId and type='search'`*
        ![omega_coredetails_api]
        - After parsing the response, `HashMap<Integer, HashMap<String, String>> getCoreDetails_hm()` method of `com.iss.esg.omega.OEntityDetailParser` is called, which forms and returns the HashMap as shown below : 
        ![omega_coredetails_hashmap_formation]
        - Now, after fetching data for all the batches, the HashMap is returned back to `getFactorValues_S` method.
    + Now the HashMap, which we got from `getFactorValues_coreDetails` is iterated and the entityId from it is checked if it exists in entities *TreeMap (<EntitiyId, SEntityData>)*
    + If its exists then, we set the *EntityName, ISIN and Country Code* into SEntityData for that entityId.
    ![getFactorValues_S_ODataAccess]
- Now, we are back into `LOAD_entities_CACHE` method, where add all the entities which we got from ODataAccess.getFactorValues() method into **`entities_CACHE(TreeMap<Integer, SEntityData>)`**


### Load Sequences Cache

- `boolean LOAD_sequences_CACHE(int sequenceID)` method of `com.iss.esg.sequence.SDataAccess` class is called and sequenceId is passed as an argument
- Inside that, `SSequenceData getSequenceData(int sequenceID)` method of the same class is called, which calls SP : **`ssp_seq_getSequenceData`**
    ```
    @_SequenceID=27
    select s.SequenceID, s.SequenceName, s.Description, s.UniverseID, s.ReleaseID, s.JobID, s.TableName, s.SequenceID_source, c.CalculationID, c.Description, c.FactorName, c.DataType, c.universeType, c.CalculationOrder, f.FormulaID, f.Description, f.Formula, r.FactorID, r.Description, r.FactorName, fx.Label, r.DataType, r.FactorGroupType, r.FactorGroupID, r.CaseQueryID 
        from S_SequenceXref x 
        inner join S_Sequences s on s.SequenceID = x.SequenceID
        inner join S_Calculations c on c.CalculationID = x.CalculationID
        inner join S_Formulas f on f.FormulaID = x.FormulaID
        inner join S_FormulaXref fx on fx.FormulaID = f.FormulaID
        inner join S_Factors r on r.FactorID = fx.FactorID
        where x.SequenceID = @_SequenceID 
        order by c.CalculationOrder, c.FactorName
    ```
    ![sequence_calculation]
    ![sequence_factors]

- This all data is stored into **SSequenceData** object, which internally holds all the sequence related information and *`ArrayList<SCalculationData>`*
    + SCalculationData internally holds all the calculation related data and *`SFormulaData`*
    + SFormulaData internally holds all the formula related data and **`TreeMap<String, SFactorData>`**, where the key is the label, which comes from SFactorData
    + SFactorData internally, holds all the factor related information
- Now, we are back inside `LOAD_sequences_CACHE` method, where we just put the SSequenceData object into **`sequences_CACHE(<sequenceId, SSequenceData>)`**


### Load Factor Values Cache

- `boolean LOAD_factorValues_CACHE(int sequenceID, TreeSet<Integer> entityIDs)` method of `com.iss.esg.sequence.SDataCache` class is called, where sequenceId and TreeSet of all the entityIds(core + Derived) is passed as an argument
- Inside it, `TreeMap<Integer, TreeMap<String, String>> getFactorValues_optimizer(TreeSet<Integer> entityIDs)` method of `com.iss.esg.sequence.SUtil` class is called which basically adds a `new TreeMap<String, String`> for each entityId and returns the `TreeMap<EntityId, new TreeMap<String, String>>`
- Now, inside it calls `TreeSet<String> getFactors_ESG(int sequenceID)` method of `com.iss.esg.sequence.SDataAccess` class is called, which calls `TreeSet<String> getFactors_ESG(int sequenceID, String storedProcedure)` method of the same class
    + It calls SP : **`ssp_seq_getFactors_ESG`**
        ```

        @_SequenceID = 27
        select distinct r.FactorName
        from S_SequenceXref x 
        inner join S_FormulaXref fx on fx.FormulaID = x.FormulaID
        inner join S_Factors r on r.FactorID = fx.FactorID and r.FactorGroupType = 'ESG'
        where x.SequenceID = @_SequenceID
        order by r.FactorName
        ```
        ![esg_factor_names]
- Now, it calls `HashMap<Integer, HashMap<String, String>> getFactorValues_S(TreeSet<String> factorNames)` method of `com.iss.esg.SDataAccess` class with the above factor names as input.
- Now the whole process is same as [Getting Universe from ESG\_Services](#getting-universe-from-esg_services), just the difference is of the SP which is `ssp_seq_getFactors_ESG` in this case. So only the factorNames supplied to `getFactorValues_S` are different.
- After getting done with `getFactorValues_S`, now, `TreeMap<Integer, TreeSet<String>> getFactors_E3(int sequenceID)` is called which internally calls `TreeMap<Integer, TreeSet<String>> getFactors_E3(int sequenceID, String storedProcedure)` with 
    + SP : **`ssp_seq_getFactors_E3`**
      ```

      @_SequenceID = 26
        select distinct r.FactorGroupID, r.FactorName
          from S_SequenceXref x 
          inner join S_FormulaXref fx on fx.FormulaID = x.FormulaID
          inner join S_Factors r on r.FactorID = fx.FactorID and r.FactorGroupType = 'E3' and r.FactorGroupID != 0 and r.CaseQueryID = 0
          where x.SequenceID = @_SequenceID
          order by r.FactorGroupID, r.FactorName
          ```

    ![e3_factor_names]

- Now, it calls `HashMap<Integer, HashMap<String, String>> getFactorValues_S(TreeMap<Integer, TreeSet<String>> factorGroups)` method of `com.iss.esg.ethix.EDataAccess` class with the above factor names as input.
- Now the whole process is same as [Getting Universe from E3](#getting-universe-from-e3), just the difference is of the SP which is `ssp_seq_getFactors_E3` in this case. So only the factorNames supplied to `getFactorValues_S` are different.
- Now, `ArrayList<SCaseData> getCaseValues_S(ArrayList<SCaseQueryData> caseQueries)` method of class `com.iss.esg.ethix.EDataAccess` is called, where input given to it by calling:         
  + `ArrayList<SCaseQueryData> getCases_E3(int sequenceID)` method of class `com.iss.esg.sequence.SDataAccess`. It calls SP : **`ssp_seq_getCases_E3`**
    ```

    @_SequenceID=26
    select distinct r.CaseQueryID, cq.CaseQuery, cq.Description, r.FactorName
	from S_SequenceXref x 
	inner join S_FormulaXref fx on fx.FormulaID = x.FormulaID
	inner join S_Factors r on r.FactorID = fx.FactorID and r.FactorGroupType = 'E3' and r.FactorGroupID = 0 and r.CaseQueryID != 0
	inner join S_CaseQueries cq on cq.CaseQueryID = r.CaseQueryID
	where x.SequenceID = @_SequenceID
	order by r.CaseQueryID, r.FactorName
    ```
    ![e3_cases]

    + Now, the values from resultSet is extracted and is put into *SCaseQueryData*

- Now inside `getCaseValues_S` method, the *ArrayList<SCaseQueryData>* is iterated and for every 
*SCaseQueryData* object and `ArrayList<SCaseData> getCaseValues_S(SCaseQueryData cqd)` method of the same class is called, which calls
  + SP : *from the SCaseQueryData object*
  + For e.g : **`ssp_seq_getCases_1`**
    ```

    select distinct m.ISSCompanyID as EntityID, e.AgentID, e.EventID, c.EthixCaseID as CaseID_ethix, c.MemexCaseID as CaseID_memex, e.SectorType_Area, e.EventTitle, e.Location, e.Signal, e.Score, e.KeyThematicArea, e.CorporateInvolvement, e.EventDate
	from Ethix_NBS_Events e 
	inner join ESG_Mapping m on m.EthixAgentID = e.AgentID and m.ISSCompanyID != 0
	inner join Ethix_Case c on c.CurrentEventID = e.EventID and c.Active = 1
	where e.Active = 1 and e.signal in ('Amber', 'RED') and e.CorporateInvolvement = 'Direct involvement' and (
	e.KeyThematicArea like '%Accounting / disclosure standards%' or
	e.KeyThematicArea like '%Anti-competitive behaviour%' or
	e.KeyThematicArea like '%Bribery%' or
	e.KeyThematicArea like '%Child labour%' or
	e.KeyThematicArea like '%Forced labour%' or
	e.KeyThematicArea like '%Labour standards%' or
	e.KeyThematicArea like '%Money laundering%' or
	e.KeyThematicArea like '%Taxes%' or
	e.KeyThematicArea like '%Union rights%' or
	e.KeyThematicArea like '%Workplace discrimination%')
	order by EntityID, e.EventID
    ```
    ![e3_case_queries_ssp_seq_getCases_1]

  + Now, all the values are set into *SCaseData* object and *`ArrayList<SCaseData>`* is returned.

- Now all the factor values from ESG_Services, E3 and E3_Case_Queries are put into `TreeMap<Integer, TreeMap<String, String>> factorValues` by `putAll_factorValues(TreeMap<Integer, TreeMap<String, String>> tm1, HashMap<Integer, HashMap<String, String>> hm2)` of `com.iss.esg.sequence.SUtil` class.
- Finally, the factorValues Hashmap is put into `factorValues_CACHE`.
- All the caches like, Entities_Cache, Sequence_Cache and Factor_Values are now loaded.

## Actual Process Call

- After loading the all the caches, `ArrayList<SEntityData> getFactorValues(int sequenceID, TreeSet<Integer> entityIDs, int threadCount)` method of `com.iss.esg.sequence.SDataAccess` class is called.
- Inside this, Executors Service is intialized with threadcount of 5.
- Now the entityIDs TreeSet is iterated, and for every entityId, SRunnable object is intialized and `call()` method is called by the worker thread.
- Inside call() method, `static SEntityData process(int sequenceID, int entityID)` method of `com.iss.esg.sequence.SProcessor` classs is called.
  ![srunnable_call]
  
- Inside `process` method, data from all the caches is loaded and `process` method of `SSequenceData` is called.
  ![sprocessor_process]

- Inside `process` method of `SSequenceData`, the *`ArrayList<SCalculationData>`* is iterated and for each calculation, and for every calculation its `process` method is called, where **SEntityData** is passed as an argument.

### Process method of SCalculationData

- 1st its checked, the entity passed to it whether its from Derived Universe or Core Universe.
  
  #### Derived Universe Process

  - `boolean HAS_DATA(SEntityData ed)` method of `SCalculationData` class is called, which checks if the **SFormulaData** is null or not.
  - If its not null, then it iterates over **SFormulaData's factors's values**.
  - It calls `Object getFactorValue_Object(String factorName)` on **SEntityData**, where it checks if the factorValue is Double or its String, if the factorValue is null then, 'NotCollected' is returned
    ![sentitydata_getFactorValue_Object]
  - After that, its checked that the value obtained, should not contains *NOTs* in it, if its true then HAS_DATA returns true.
   ![scalculationdata_has_data]
  - If `HAS_DATA()` method returns true, then **SFormulaData's factors's values** are iterated, so for every SFactorData **SEntityData's** `Object getFactorValue_Object(String factorName)` method is called which gives the value for the factorName.
  - Now, this obtained value is put into `TreeMap<String, Object> values` where, key of the map is the label which we get from SFactorData and value of the map is the obtained value.
  - Now, its checked whether the description of SCalculationData is *'Legacy...'*
  - If its Legacy, then ***SCalculationUtil's** `String evaluate(TreeMap<String, Object> values, String formula)`*
  - Else ***SCalculationUtil2's** `String evaluate(TreeMap<String, Object> values, String formula)`*
  - Inside *SCalculationUtil's evaluate* method :
    + `String REPLACE(String s)` method of the same class is called, which replaces the key in the given string with the value from *REPLACE's Map*
        ![scalculationutil_replace]
    + *values Map* is iterated and for each label is replaced with its appropriate value
    + After iteration is done, `String RESTORE(String s)` method of the same class is called
        ![scalculationutil_restore]
    + After that *formula String* is checked whether it contains **'MEDIAN('** in it, if it contains then,
      - `String process_MEDIAN(String formula)` of the same class is called
      ![scalculationutil_process_median]
    + After that, the final formula is set into `WorkBook CELL`, and then 
     `String evaluate()` method of the same class is called, which evalutes the formula and returns the calculated value.
      ![scalculationutil_evaluate]
    + The calculated value is returned back to `process(SEntityData ed)` method of `SCalculationData` class.
  + Inside *SCalculationUtil2 evaluate* method :
    - WorkBook is initialized, Sheet is created, Row is created at 0th index
    - Now the values map is iterated, for every lable we get a column Index from `SUtil's getCellID(String cellCode)` method. After getting the Column Index, we create the cell and then set the value in the cell from *values's map*
    - After iteration is completed, another row is created at index 1 and a cell is created and 0th column Index, and formula is set into it.
    - After evaluating the formula, we get the calculated value and its returned back to `process(SEntityData ed)` method of `SCalculationData` class.
  + The returned value from *SCalculationUtil's evaluate* method is checked for null and if its not null then its added into `SEntityData's factorValues Map ed.addFactorValue(factorName, factorValue);`	
  + If `HAS_DATA()` returns false then 'NotCollected' is added in `SEntityData's factorValues Map ed.addFactorValue(factorName, "NotCollected");`
  
  #### Core Universe Process
  - Its exactly same as [Derived Universe Process](#derived-universe-process), just the difference is it does not call `HAS_DATA(SEntityData ed)` method of SCalculationData
- The entity Id, which does not belong in either Derived Universe or Core Universe, for that, *'NotCollected'* value is set into SEntityData `ed.addFactorValue(factorName, "NotCollected")`


- Now Back to **Actual Process Call** method `ArrayList<SEntityData> getFactorValues(int sequenceID, TreeSet<Integer> entityIDs, int threadCount)`, here the Furture Objects are iterated and added to ArrayList<SEntityData>
  ```
  for (int i = 0; i < entities_FUTURE.size(); i++)
        {
          ed = (SEntityData)entities_FUTURE.get(i).get();
          
          if (ed != null)
          {
            entities.add(ed);
          }
        }
  ```
- This ArrayList is now returned to `ArrayList<SEntityData> getFactorValues(int sequenceID)` method of `com.iss.esg.sequence.SDataAccess` class.
- From here the control is returned back to `SServlet` class

  #### Misc.

  - SCalculationUtil's static method block:
  - Workbook instance is initialized, Cell is initialized and Cell Type is set as Formula
  - Also **REPLACE** and **RELOAD**, these two `TreeMap<String, String>` are also initialized
  - REPLACE : 
      + `TreeMap<String, String> LOAD_REPLACE()` method is called, which internally calls `ArrayList<String> getFunctions()` method of the same class.
      ![scalculationutil_loadreplace]

        ![scalculationutil_getfunctions]

  - RESTORE : 
      + `TreeMap<String, String> LOAD_RESTORE()` method is called, which internally calls `ArrayList<String> getFunctions()` method of the same class.
      ![scalculationutil_loadrestore]


## Excel Creation

- We get the excel workbook by calling `SXSSFWorkbook getWorkbook(int sequenceID, ArrayList<SEntityData> entities)` method of `com.iss.esg.sequence.SFileAccess` class.
- Inside that, we get the SSequenceData from Sequence Cache and then call `SXSSFWorkbook getWorkbook(SSequenceData sd, ArrayList<SEntityData> entities)` of the same class
![sfileaccess_getWorkbook]
    + Inside that, we call `SXSSFWorkbook getWorkbook(SSequenceData sd)` of the same class
    + Inside that, we get the template name from `SUtil.getFileName_template()`
    + XSSFWorkbook is created with the excel template
    + From that, we get sheet at 0th index, which is the 1st sheet named *'sequence...'*
    + This sheet has header as EntityID, EntityName, ISIN, CountryCode, 1, 2, 3, 4, 5,........., 100
    ![excel_sequence_sheet]
    + Now, `ArrayList<SFactorData> getFactors_sequence()` method is called on SSequenceData object to get List of SFactorData
        ![ssequencedata_getfactors_sequence]
    + Now, List of SFactorData is iterated and starting from 1st row **column 5** *(index 4)*, headers are added, which are the factor names
    + After adding the headers(fator names), extra column headers are removed.
    + Now sheet2 named *input...* is taken, which is same as sheet1, it also has the same headers as sheet1
    + Now, `ArrayList<SFactorData> getFactors_input()` method is called on SSequenceData object to get List of SFactorData
        
        ![ssequencedata_getfactors_input]
        - *Note:  Here, when getFactors_input() is called on SFormulaData object, then this method only gets the factors whose **factorGroupType** is **'ESG' or 'E3'***
    + Now, similar to sheet 1, here also the list of SFactorData is iterated, headers are set with factornames and the extra column headers are removed.
    + Now, the workbook is returned back to `SXSSFWorkbook getWorkbook(SSequenceData sd, ArrayList<SEntityData> entities)` method
- Now, `void getWorkbook_sheet(SXSSFWorkbook workbook, int sheetID, ArrayList<SFactorData> factors, ArrayList<SEntityData> entities)` method is called 2 times, 1st time for sheet 1(sequence...) with **Calculation Factors** *(which we get from getFactors_sequence())* and 2nd time for sheet 2(input...) with **Actual Factors** *(which we get from getFactors_input())*.
- In this method, SEntityData List is iterated and inside that SFactorData List is iterated, so rows and cells are created and data is put into the cells also cell styling is done based on the data types.
- Now, `void getWorkbook_sheet(SXSSFWorkbook workbook, int sheetID, SSequenceData sd)` method is called for the *last sheet (**cases...**)*
- In this method, first it checks do we have cases in th sequence, it is checked by calling
  `boolean CASES()` method on SSequenceData object
    + Inside that, `ArrayList<SFactorData> getFactors_cases()` method of the same class is called which iterates on the calculations and from every SCalculationData object it takes out the SFormulaData object and on that it calls `TreeMap<String, SFactorData> getFactors_cases()` method which iterates on the factors.values() that means it iterates on SFactorData and if SFactorData object's FactorGroupType is E3 and its factorGroupID = 0 and its caseQueryID !=0, then it adds that factor in the map and returns the map
    + In the `boolean CASES()` method, the size of the returned `ArrayList<SFactorData>` from `getFactors_cases()` is checked, if its greater than 0 then true is returned, else false is returned
- If `CASES()` gives us true then we get the *cases...* sheet from the workbook and put data into it, else we remove the *cases...* sheet from the workbook.
- Now we call, `ArrayList<SCaseQueryData> getCases_E3(int sequenceID)` method of `com.iss.esg.sequence.SDataAccess` which calls SP : **`ssp_seq_getCases_E3`**
  ```
  @_SequenceID = 26
  select distinct r.CaseQueryID, cq.CaseQuery, cq.Description, r.FactorName
    from S_SequenceXref x 
    inner join S_FormulaXref fx on fx.FormulaID = x.FormulaID
    inner join S_Factors r on r.FactorID = fx.FactorID and r.FactorGroupType = 'E3' and r.FactorGroupID = 0 and r.CaseQueryID != 0
    inner join S_CaseQueries cq on cq.CaseQueryID = r.CaseQueryID
    where x.SequenceID = @_SequenceID
    order by r.CaseQueryID, r.FactorName
  ```
  ![e3_cases]
- Now, the values from resultSet is extracted and is put into *SCaseQueryData*, so this method returns `ArrayList<SCaseQueryData>`
- The ArrayList obtained above is feeded to `ArrayList<SCaseData> getCaseValues_S(ArrayList<SCaseQueryData> caseQueries)` method of `com.iss.esg.ethix.EDataAccess` class, which iterates over this list and for each *SCaseQueryData* object, `ArrayList<SCaseData> getCaseValues_S(SCaseQueryData cqd)` method of the same class is called, which executes the mentioned SP in *SCaseQueryData* object and its resultset is extracted and put into *SCaseData* object, so we get a list of *SCaseData*.
- Now, all these lists are added into one list and now this list is iterated in `getWorkbook_sheet` method, where the data from *SCaseData* is extraced and put into excel sheet.
- In this way, the last sheet of the workbook is also filled.
- Finally, the Workbook is returned back to SServlet and written in the response.

[universe_type_T]: ./img/Universe_Type_T.png
[universe_type_I]: ./img/Universe_Type_I.png
[universe_type_E]: ./img/Universe_Type_E.png
[omega_indexes_api]: ./img/Omega_Indexes_API.PNG
[esg_derived_factor_names]: ./img/ESG_Derived_FactorNames.PNG
[factorgroups_entities_api]: ./img/FactorGroups_Entities_API.PNG
[e3_derived_factor_names]: ./img/E3_Derived_FactorNames.PNG
[e3_entityId_factornames_factorvalues]: ./img/E3_EnityID_FactorNames_FactorValues_API.PNG
[sdataaccess_getFactorValues]: ./img/SDataAccess_getFactorValues_Main.PNG
[omega_coredetails_api]: ./img/Omega_GetCoreDetails.PNG
[omega_coredetails_hashmap_formation]: ./img/Omega_CoreDetailsResponse_HashMap.PNG
[getFactorValues_S_ODataAccess]: ./img/getFactorValues_S_ODataAccess.PNG
[sequence_calculation]: ./img/Sequence_Calculation.PNG
[sequence_factors]: ./img/Sequence_Factos.PNG
[esg_factor_names]: ./img/ESG_FactorNames.PNG
[e3_factor_names]: ./img/E3_FactorNames.PNG
[e3_cases]: ./img/E3_Cases.PNG
[e3_case_queries_ssp_seq_getCases_1]: ./img/E3_CaseQueries_SP_ssp_seq_getCases_1.PNG
[srunnable_call]: ./img/SRunnable_call.PNG
[sprocessor_process]: ./img/SProcessor_process.PNG
[sfileaccess_getWorkbook]: ./img/SFileAccess_getWorkbook.PNG
[excel_sequence_sheet]: ./img/Excel_Sequence_Sheet.PNG
[sentitydata_getFactorValue_Object]: ./img/SEntityData_getFactorValue_Object.PNG
[scalculationdata_has_data]: ./img/SCalculationData_HAS_DATA.PNG
[scalculationutil_replace]: ./img/SCalculationUtil_REPLACE.PNG
[scalculationutil_restore]: ./img/SCalculationUtil_RESTORE.PNG
[scalculationutil_process_median]: ./img/SCalculationUtil_process_MEDIAN.PNG
[scalculationutil_evaluate]: ./img/SCalculationUtil_evaluate.PNG
[scalculationutil_loadreplace]: ./img/SCalculationUtil_LOAD_REPLACE.PNG
[scalculationutil_loadrestore]: ./img/SCalculationUtil_LOAD_RESTORE.PNG
[scalculationutil_getfunctions]: ./img/SCalculationUtil_getFunctions.PNG
[ssequencedata_getfactors_sequence]: ./img/SSequenceData_getFactors_sequence.PNG
[ssequencedata_getfactors_input]: ./img/SSequenceData_getFactors_input.PNG
