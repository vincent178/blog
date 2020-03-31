---
title: "Database"
date: 2020-03-30T14:56:44+08:00
draft: true
---

Seq Scan

Index Scan

Index Only Scan
return data all in index, so PG don't need to check the table again

Bitmap Index Scan

select relname,last_vacuum, last_autovacuum, last_analyze, last_autoanalyze, n_dead_tup, n_live_tup from pg_stat_user_tables where relname in ('orders', 'products', 'users', 'logged_actions');

select relname,last_vacuum, last_autovacuum, last_analyze, last_autoanalyze, n_dead_tup, n_live_tup from pg_stat_user_tables order by last_analyze desc limit 10;

select relname,last_vacuum, last_autovacuum, last_analyze, last_autoanalyze, n_dead_tup, n_live_tup from pg_stat_user_tables where relname = 'logged_actions';


SELECT name, setting FROM pg_settings WHERE name in ('autovacuum', 'log_autovacuum_min_duration', 'autovacuum_max_workers', 'autovacuum_vacuum_scale_factor', 'autovacuum_analyze_scale_factor');


SELECT relname, relkind, reltuples, relpages FROM pg_class WHERE relname LIKE 'orders%' OR relname LIKE 'index_orders%' OR relname = 'idx_orders_purchased_at_not_null';

SELECT relname, relkind, reltuples, relpages FROM pg_class WHERE relname LIKE 'products%' OR relname LIKE 'index_products%' OR relname in ('idx_products_ext_product_id_is_not_null', 'idx_user_id_and_products_ext_tag_is_not_null', 'idx_user_id_and_products_ext_tag_is_not_null', 'product_by_template_status_condition_size', 'product_by_template_status_condition_size_us');


explain analyse SELECT COUNT(*) FROM "orders" INNER JOIN "products" ON "products"."id" = "orders"."product_id" WHERE "orders"."paid_out" = TRUE AND "products"."user_id" = 5329506;
explain analyse SELECT COUNT(*) FROM "orders" INNER JOIN "products" ON "products"."id" = "orders"."product_id" WHERE "products"."user_id" = 5329506 ;


explain analyse SELECT COUNT(*) FROM "orders" INNER JOIN "products" ON "products"."id" = "orders"."product_id" WHERE "orders"."paid_out" = TRUE AND "products"."user_id" = 5329506 AND "orders"."purchased_at"||'' IS NOT NULL;

select email from users where id = 5329506;

select count(*) from orders where purchased_at is not null;

select count(*) from orders where 


explain analyse select id from products where user_id = 5329506;
explain analyse select count(*) from orders where "orders"."paid_out" = TRUE and product_id = 1234;
explain analyse select count(*) from orders where "orders"."paid_out" = TRUE AND product_id in (select id from products where user_id = 5329506);

https://www.depesz.com/2013/04/27/explaining-the-unexplainable-part-2/#bitmap-index-scan

* https://medium.com/compass-true-north/postgres-at-scale-query-performance-and-autovacuuming-for-large-tables-d7e8ad40b16b
* https://medium.com/compass-true-north/dealing-with-significant-postgres-database-bloat-what-are-your-options-a6c1814a03a5
* https://github.com/Vonng/pg/blob/master/admin/tune-hot\_standby\_feedback.md
* https://www.postgresql.org/docs/9.0/storage-page-layout.html
* https://www.datadoghq.com/blog/postgresql-vacuum-monitoring/
* https://www.datadoghq.com/blog/postgresql-monitoring/
* https://www.datadoghq.com/blog/monitoring-101-collecting-data/
* https://www.depesz.com/2013/04/27/explaining-the-unexplainable-part-2/#bitmap-index-scan