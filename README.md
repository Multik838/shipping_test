# Проект 2
Опишите здесь поэтапно ход решения задачи. Вы можете ориентироваться на тот план выполнения проекта, который мы предлагаем в инструкции на платформе.


### 0. Создание исходной таблицы `Shipping`
```SQL
DROP TABLE IF EXISTS public.shipping;
CREATE TABLE public.shipping(
   ID serial ,
   shippingid                         BIGINT,
   saleid                             BIGINT,
   orderid                            BIGINT,
   clientid                           BIGINT,
   payment_amount                     NUMERIC(14,2),
   state_datetime                     TIMESTAMP,
   productid                          BIGINT,
   description                        text,
   vendorid                           BIGINT,
   namecategory                       text,
   base_country                       text,
   status                             text,
   state                              text,
   shipping_plan_datetime             TIMESTAMP,
   hours_to_plan_shipping             NUMERIC(14,2),
   shipping_transfer_description      text,
   shipping_transfer_rate             NUMERIC(14,3),
   shipping_country                   text,
   shipping_country_base_rate         NUMERIC(14,3),
   vendor_agreement_description       text,
   PRIMARY KEY (ID)
);
CREATE INDEX shippingid ON public.shipping (shippingid);
COMMENT ON COLUMN public.shipping.shippingid is 'id of shipping of sale';
```
```SQL
COPY shipping (shippingid,saleid,orderid,clientid,payment_amount,state_datetime,productid,description,vendorid,namecategory,base_country,status,state,shipping_plan_datetime,hours_to_plan_shipping,shipping_transfer_description,shipping_transfer_rate,shipping_country,shipping_country_base_rate,vendor_agreement_description)
FROM 'E:\Downloads\Browser\shipping.csv' 
DELIMITER ',' 
CSV HEADER;

--исправление орфографической ошибки
UPDATE shipping
SET state = REPLACE(state,'recieved','received')
WHERE state='recieved';
```
### 1. Создание таблицы `shipping_country`
```SQL
DROP TABLE IF EXISTS public.shipping_country;
CREATE TABLE public.shipping_country (
shipping_country_id serial,
shipping_country text,
shipping_country_base_rate numeric(14,3),
PRIMARY KEY (shipping_country_id));
```
```SQL
INSERT INTO public.shipping_country (shipping_country,shipping_country_base_rate)
SELECT shipping_country,shipping_country_base_rate
FROM shipping
GROUP BY shipping_country,shipping_country_base_rate;
```
### 2. Создание таблицы `shipping_agreement`
```SQL
DROP TABLE IF EXISTS public.shipping_agreement;
CREATE TABLE public.shipping_agreement (
agreementid serial,
agreement_number text,
agreement_rate NUMERIC(14,3),
agreement_commission NUMERIC(14,3),
PRIMARY KEY (agreementid));
```
```SQL
INSERT INTO shipping_agreement
SELECT DISTINCT
	(regexp_split_to_array(vendor_agreement_description, ':'))[1]::integer,
	(regexp_split_to_array(vendor_agreement_description, ':'))[2]::text,
	(regexp_split_to_array(vendor_agreement_description, ':'))[3]::NUMERIC(14,3),
	(regexp_split_to_array(vendor_agreement_description, ':'))[4]::NUMERIC(14,3)
FROM shipping
ORDER BY (regexp_split_to_array(vendor_agreement_description, ':'))[1]::integer;
```
### 3. Создание таблицы `shipping_transfer`
```SQL
DROP TABLE IF EXISTS public.shipping_transfer;
CREATE TABLE public.shipping_transfer (
transfer_type_id serial,
transfer_type text,
transfer_model text,
shopping_transfer_rate NUMERIC(14,3),
PRIMARY KEY (transfer_type_id));
```
```SQL
INSERT INTO public.shipping_transfer (transfer_type,transfer_model,shopping_transfer_rate)
SELECT DISTINCT
	(regexp_split_to_array(shipping_transfer_description, ':'))[1]::text,
	(regexp_split_to_array(shipping_transfer_description, ':'))[2]::text,
	shipping_transfer_rate
FROM shipping;
```
### 4. Создание таблицы `shipping_info`
```SQL
DROP TABLE IF EXISTS public.shipping_info;
CREATE TABLE public.shipping_info (
shippingid bigint,
vendorid bigint,
payment_amount NUMERIC(14,2),
shipping_plan_datetime timestamp,
transfer_type_id bigint,
shipping_country_id bigint,
agreementid bigint,
PRIMARY KEY (shippingid),
FOREIGN KEY (transfer_type_id) REFERENCES shipping_transfer (transfer_type_id),
FOREIGN KEY (shipping_country_id) REFERENCES shipping_country (shipping_country_id),
FOREIGN KEY (agreementid) REFERENCES shipping_agreement (agreementid)
);
```
```SQL
INSERT INTO shipping_info
SELECT
	main_table.shippingid,
	main_table.vendorid,
	main_table.payment_amount,
	main_table.shipping_plan_datetime,
	shipping_transfer.transfer_type_id,
	shipping_country.shipping_country_id,
	shipping_agreement.agreementid
FROM (
	SELECT DISTINCT
		shippingid,
		vendorid,
		payment_amount,
		shipping_plan_datetime,
		shipping_transfer_description,
		shipping_country,
		vendor_agreement_description
	FROM shipping
	) AS main_table
LEFT JOIN shipping_transfer
ON 
	(regexp_split_to_array(main_table.shipping_transfer_description, ':'))[1]::text=shipping_transfer.transfer_type AND
	(regexp_split_to_array(main_table.shipping_transfer_description, ':'))[2]::text=shipping_transfer.transfer_model
LEFT JOIN shipping_country ON main_table.shipping_country=shipping_country.shipping_country
LEFT JOIN shipping_agreement
ON
	(regexp_split_to_array(main_table.vendor_agreement_description, ':'))[2]::text=shipping_agreement.agreement_number AND
	(regexp_split_to_array(main_table.vendor_agreement_description, ':'))[3]::NUMERIC(14,3)=shipping_agreement.agreement_rate AND
	(regexp_split_to_array(main_table.vendor_agreement_description, ':'))[4]::NUMERIC(14,3)=shipping_agreement.agreement_commission
ORDER BY main_table.shippingid;
```
### 5. Создание таблицы `shipping_status`
```SQL
DROP TABLE IF EXISTS public.shipping_status;
CREATE TABLE public.shipping_status (
shippingid bigint,
status text,
state text,
shipping_start_fact_datetime timestamp,
shipping_end_fact_datetime timestamp
--CONSTRAINT fk_shippingid FOREIGN KEY (shippingid) REFERENCES shipping (shippingid)
	);

--CREATE INDEX shippingid ON public.shipping_status (shippingid);

COMMENT ON COLUMN public.shipping_status.status is 'статус доставки в таблице shipping по данному shippingid. Может принимать значения in_progress — доставка в процессе, либо finished — доставка завершена.';
COMMENT ON COLUMN public.shipping_status.state is 'промежуточные точки заказа, которые изменяются в соответствии с обновлением информации о доставке по времени state_datetime';
```
```SQL
INSERT INTO shipping_status
SELECT 
	main_table.shippingid, 
	shipping.status,
	shipping.state,
	booked_state.shipping_start_fact_datetime,
	received_state.shipping_end_fact_datetime
FROM (
	SELECT 
		shippingid,
		MAX(state_datetime) AS state_datetime
	FROM shipping
	GROUP BY shippingid
	) AS main_table
LEFT JOIN shipping ON main_table.shippingid=shipping.shippingid AND main_table.state_datetime=shipping.state_datetime
LEFT JOIN (
	SELECT shippingid,state_datetime AS shipping_start_fact_datetime
	FROM shipping
	WHERE state='booked'
	) AS booked_state
ON main_table.shippingid=booked_state.shippingid
LEFT JOIN (
	SELECT shippingid,state_datetime AS shipping_end_fact_datetime
	FROM shipping
	WHERE state='received'
	) AS received_state
ON main_table.shippingid=received_state.shippingid
ORDER BY main_table.shippingid;
```
### 6. Создание представления `shipping_datamart`
```SQL
DROP VIEW IF EXISTS shipping_datamart;
CREATE VIEW shipping_datamart AS
SELECT 
	shipping_info.shippingid,
	shipping_info.vendorid,
	shipping_transfer.transfer_type,
	DATE_PART('DAY', AGE(shipping_status.shipping_end_fact_datetime,shipping_status.shipping_start_fact_datetime)) AS full_day_at_shipping,
	(CASE WHEN shipping_status.shipping_end_fact_datetime>shipping_info.shipping_plan_datetime THEN 1 ELSE 0 END) AS is_delayed,
	(CASE WHEN shipping_status.status='finished' THEN 1 ELSE 0 END) AS is_shipping_finish,
	(CASE WHEN shipping_status.shipping_end_fact_datetime>shipping_info.shipping_plan_datetime THEN DATE_PART('DAY', AGE(shipping_status.shipping_end_fact_datetime,shipping_info.shipping_plan_datetime)) ELSE 0 END) AS delay_day_at_shipping,
	shipping_info.payment_amount,
	shipping_info.payment_amount * (shipping_country.shipping_country_base_rate + shipping_agreement.agreement_rate + shipping_transfer.shopping_transfer_rate) AS vat,
	shipping_info.payment_amount * shipping_agreement.agreement_commission AS profit
FROM shipping_info
LEFT JOIN shipping_transfer ON shipping_info.transfer_type_id=shipping_transfer.transfer_type_id
LEFT JOIN shipping_status ON shipping_info.shippingid=shipping_status.shippingid
LEFT JOIN shipping_country ON shipping_info.shipping_country_id=shipping_country.shipping_country_id
LEFT JOIN shipping_agreement ON shipping_info.agreementid=shipping_agreement.agreementid;
```
