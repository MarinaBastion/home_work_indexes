# Создаем индекс на поле name в таблице Dilers

create index on dbo."Dilers"(name); 

explain (costs, verbose,  analyze) --, analyze

select * from dbo."Dilers" where name ilike '%alex%'

Seq Scan on dbo."Dilers"  (cost=0.00..31.60 rows=314 width=47) (actual time=0.013..1.617 rows=320 loops=1)
# индекс не подхватился

explain (costs, verbose, analyze) --, analyze

select name from dbo."Dilers" where name = 'Alex Green'

Index Only Scan using "Dilers_name_idx" on dbo."Dilers"  (cost=0.28..4.31 rows=2 width=23) (actual time=0.163..0.165 rows=2 loops=1)

# индекс использован

# полнотекстовый поиск 

alter table  dbo."Dilers" add column diler_lexeme tsvector;
update dbo."Dilers"
set diler_lexeme = to_tsvector(name)

select name, adress from dbo."Dilers" where diler_lexeme @@ to_tsquery('jone' ) limit 10;

explain select name, adress from dbo."Dilers" where diler_lexeme @@ to_tsquery('jone' ) ; 

CREATE INDEX search_index_name ON dbo."Dilers" USING GIN (diler_lexeme);
 
explain select name, adress from dbo."Dilers" where diler_lexeme @@ to_tsquery('jone' ) ; 
 
   ->  Bitmap Index Scan on search_index_name  (cost=0.00..14.65 rows=320 width=0)
   
# индекс на поле с функцией
explain analyze select  product_id, change_date, price from  dbo."Prices" where extract(year from change_date) = extract(year from now());
  
Seq Scan on "Prices"  (cost=0.00..34.60 rows=6 width=16) (actual time=0.032..1.288 rows=198 loops=1)

CREATE INDEX i_change_date_year ON  dbo."Prices" (extract(year from change_date));
  
explain analyze select  product_id, change_date, price from  dbo."Prices" where extract(year from change_date) = extract(year from now());
  
Bitmap Index Scan on i_change_date_year  (cost=0.00..4.33 rows=6 width=0) (actual time=0.114..0.114 rows=198 loops=1)
  
# стоимость выборки сократилась с 34.60  до 4.33  
  
# создание составного индекса

INSERT INTO dbo."Delivers"(
diler_id, store_id, product_id, deliver_date)
select d.id, s.id, p.id, now() - interval '4 MONTH'
from  dbo."Dilers" as d cross join  dbo."Stores" as s 
cross join  dbo."Products" p 
    
explain (costs, verbose, format json, analyze)   select * from   dbo."Delivers" where diler_id = 999 and  store_id = 3
[
{
"Plan": {
    "Node Type": "Append",
    "Parallel Aware": false,
    "Async Capable": false,
    "Startup Cost": 0,
    "Total Cost": 880.39,
    "Plan Rows": 11,
    "Plan Width": 24,
    "Actual Startup Time": 0.767,
    "Actual Total Time": 7.956,
    "Actual Rows": 9,
    "Actual Loops": 1,
    "Subplans Removed": 0,
    "Plans": [
    {
        "Node Type": "Seq Scan",
        "Parent Relationship": "Member",
        "Parallel Aware": false,
        "Async Capable": false,
        "Relation Name": "Delivers_1",
        "Schema": "dbo",
        "Alias": "Delivers_1",
        "Startup Cost": 0,
        "Total Cost": 33.55,
        "Plan Rows": 1,
        "Plan Width": 24,
        "Actual Startup Time": 0.006,
        "Actual Total Time": 0.006,
        "Actual Rows": 0,
        "Actual Loops": 1,
        "Output": [
        "\"Delivers_1\".id",
        "\"Delivers_1\".diler_id",
        "\"Delivers_1\".store_id",
        "\"Delivers_1\".product_id",
        "\"Delivers_1\".deliver_date"
        ],
        "Filter": "((\"Delivers_1\".diler_id = 999) AND (\"Delivers_1\".store_id = 3))",
        "Rows Removed by Filter": 0
    },
    {
        "Node Type": "Seq Scan",
        "Parent Relationship": "Member",
        "Parallel Aware": false,
        "Async Capable": false,
        "Relation Name": "Delivers_2",
        "Schema": "dbo",
        "Alias": "Delivers_2",
        "Startup Cost": 0,
        "Total Cost": 813.24,
        "Plan Rows": 9,
        "Plan Width": 24,
        "Actual Startup Time": 0.76,
        "Actual Total Time": 7.926,
        "Actual Rows": 9,
        "Actual Loops": 1,
        "Output": [
        "\"Delivers_2\".id",
        "\"Delivers_2\".diler_id",
        "\"Delivers_2\".store_id",
        "\"Delivers_2\".product_id",
        "\"Delivers_2\".deliver_date"
        ],
        "Filter": "((\"Delivers_2\".diler_id = 999) AND (\"Delivers_2\".store_id = 3))",
        "Rows Removed by Filter": 38007
    },
    {
        "Node Type": "Seq Scan",
        "Parent Relationship": "Member",
        "Parallel Aware": false,
        "Async Capable": false,
        "Relation Name": "Delivers_3",
        "Schema": "dbo",
        "Alias": "Delivers_3",
        "Startup Cost": 0,
        "Total Cost": 33.55,
        "Plan Rows": 1,
        "Plan Width": 24,
        "Actual Startup Time": 0.017,
        "Actual Total Time": 0.017,
        "Actual Rows": 0,
        "Actual Loops": 1,
        "Output": [
        "\"Delivers_3\".id",
        "\"Delivers_3\".diler_id",
        "\"Delivers_3\".store_id",
        "\"Delivers_3\".product_id",
        "\"Delivers_3\".deliver_date"
        ],
        "Filter": "((\"Delivers_3\".diler_id = 999) AND (\"Delivers_3\".store_id = 3))",
        "Rows Removed by Filter": 2
    }
    ]
},
"Planning Time": 0.248,
"Triggers": [],
"Execution Time": 8.044
}
]

CREATE INDEX delivers_ds_idx ON dbo."Delivers" (diler_id, store_id); -- кардинальность diler_id  выше чем store_id   
 
explain (costs, verbose, format json, analyze)   select * from   dbo."Delivers" where diler_id = 999 and  store_id = 3 
[
{
"Plan": {
    "Node Type": "Append",
    "Parallel Aware": false,
    "Async Capable": false,
    "Startup Cost": 0.15,
    "Total Cost": 44.58,
    "Plan Rows": 11,
    "Plan Width": 24,
    "Actual Startup Time": 0.187,
    "Actual Total Time": 0.228,
    "Actual Rows": 9,
    "Actual Loops": 1,
    "Subplans Removed": 0,
    "Plans": [
    {
        "Node Type": "Index Scan",
        "Parent Relationship": "Member",
        "Parallel Aware": false,
        "Async Capable": false,
        "Scan Direction": "Forward",
        "Index Name": "Delivers_1_diler_id_store_id_idx",
        "Relation Name": "Delivers_1",
        "Schema": "dbo",
        "Alias": "Delivers_1",
        "Startup Cost": 0.15,
        "Total Cost": 8.17,
        "Plan Rows": 1,
        "Plan Width": 24,
        "Actual Startup Time": 0.113,
        "Actual Total Time": 0.113,
        "Actual Rows": 0,
        "Actual Loops": 1,
        "Output": ["\"Delivers_1\".id", "\"Delivers_1\".diler_id", "\"Delivers_1\".store_id", "\"Delivers_1\".product_id", "\"Delivers_1\".deliver_date"],
        "Index Cond": "((\"Delivers_1\".diler_id = 999) AND (\"Delivers_1\".store_id = 3))",
        "Rows Removed by Index Recheck": 0
    },
    {
        "Node Type": "Bitmap Heap Scan",
        "Parent Relationship": "Member",
        "Parallel Aware": false,
        "Async Capable": false,
        "Relation Name": "Delivers_2",
        "Schema": "dbo",
        "Alias": "Delivers_2",
        "Startup Cost": 4.38,
        "Total Cost": 35.32,
        "Plan Rows": 9,
        "Plan Width": 24,
        "Actual Startup Time": 0.072,
        "Actual Total Time": 0.097,
        "Actual Rows": 9,
        "Actual Loops": 1,
        "Output": ["\"Delivers_2\".id", "\"Delivers_2\".diler_id", "\"Delivers_2\".store_id", "\"Delivers_2\".product_id", "\"Delivers_2\".deliver_date"],
        "Recheck Cond": "((\"Delivers_2\".diler_id = 999) AND (\"Delivers_2\".store_id = 3))",
        "Rows Removed by Index Recheck": 0,
        "Exact Heap Blocks": 9,
        "Lossy Heap Blocks": 0,
        "Plans": [
        {
            "Node Type": "Bitmap Index Scan",
            "Parent Relationship": "Outer",
            "Parallel Aware": false,
            "Async Capable": false,
            "Index Name": "Delivers_2_diler_id_store_id_idx",
            "Startup Cost": 0.00,
            "Total Cost": 4.38,
            "Plan Rows": 9,
            "Plan Width": 0,
            "Actual Startup Time": 0.062,
            "Actual Total Time": 0.062,
            "Actual Rows": 9,
            "Actual Loops": 1,
            "Index Cond": "((\"Delivers_2\".diler_id = 999) AND (\"Delivers_2\".store_id = 3))"
        }
        ]
    },
    {
        "Node Type": "Seq Scan",
        "Parent Relationship": "Member",
        "Parallel Aware": false,
        "Async Capable": false,
        "Relation Name": "Delivers_3",
        "Schema": "dbo",
        "Alias": "Delivers_3",
        "Startup Cost": 0.00,
        "Total Cost": 1.03,
        "Plan Rows": 1,
        "Plan Width": 24,
        "Actual Startup Time": 0.013,
        "Actual Total Time": 0.013,
        "Actual Rows": 0,
        "Actual Loops": 1,
        "Output": ["\"Delivers_3\".id", "\"Delivers_3\".diler_id", "\"Delivers_3\".store_id", "\"Delivers_3\".product_id", "\"Delivers_3\".deliver_date"],
        "Filter": "((\"Delivers_3\".diler_id = 999) AND (\"Delivers_3\".store_id = 3))",
        "Rows Removed by Filter": 2
    }
    ]
},
"Planning Time": 1.553,
"Triggers": [
],
"Execution Time": 0.276
}
]
# общая стоимость сократилась с 880 до 44
# по первой партиции сработал Index Scan , по второй партиции срабтал Bitmap Index Scan ,
# а по третьей - Seq Scan с приминением фильтра по искомым полям