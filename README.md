```sql

create schema if not exists "monitoring";

create schema if not exists "components";

create schema if not exists "tenants";

create extension if not exists timescaledb;

drop table if exists monitoring.heart_beats;

create table monitoring.heart_beats (
	heart_beat_id bigserial NOT NULL,
	instance_id int8 NULL,
	tenant_id int8 NULL,
	insert_date_time_utc timestamp(0) NOT NULL DEFAULT timezone('utc'::text, now()),
	constraint heart_beats_pk PRIMARY KEY (heart_beat_id, insert_date_time_utc)
);

select create_hypertable('monitoring.heart_beats', 'insert_date_time_utc', chunk_time_interval => interval '12 hour');

create index on monitoring.heart_beats(insert_date_time_utc, instance_id) WITH (timescaledb.transaction_per_chunk);

drop table if exists tenants.tenants;
create table tenants.tenants (
	tenant_id bigserial,
	tenant_name character varying(255),
	tenant_description character varying(500),
	insert_date_time_utc timestamp(0) NULL DEFAULT timezone('utc'::text, now())
);

drop table if exists tenants.users;
create table tenants.users (
	user_id bigserial,
	tenant_id bigint,
	username character varying(255),
	password character varying(255),
	email character varying(255),
	first_name character varying(255),
	last_name character varying(255),
	insert_date_time_utc timestamp(0) NULL DEFAULT timezone('utc'::text, now())
);


drop table if exists tenants.users_signin;
create table tenants.users_signin (
	user_id bigserial,
	token character varying(500),
	insert_date_time_utc timestamp(0) NULL DEFAULT timezone('utc'::text, now())
);

drop table if exists components.components;
create table components.components (
	component_id bigserial,
	component_name character varying(255),
	component_description character varying(500),
	component_type smallint,
	tenant_id bigint,
	insert_date_time_utc timestamp(0) NULL DEFAULT timezone('utc'::text, now())
);


create or replace function monitoring.heart_beats_add(p_instance_id bigint, p_tenant_id bigint)
 returns void
 language plpgsql
as $function$
begin

	insert into monitoring.heart_beats(instance_id, tenant_id)
	values(p_instance_id, p_tenant_id);

end;
$function$;


create or replace function components.components_add(
	p_component_name character varying(255),
	p_component_description character varying(500),
	p_component_type smallint,
	p_tenant_id bigint)
 returns bigint
 language plpgsql
as $function$
	declare o_component_id bigint;
begin

	insert into components.components(component_name, component_description, component_type, tenant_id)
	values(p_component_name, p_component_description, p_component_type)
	returning component_id into o_component_id;

end;
$function$;



```
