
```sql

create schema if not exists "monitoring";

create schema if not exists "components";

create schema if not exists "tenants";

create extension if not exists timescaledb;

create extension if not exists pgcrypto schema tenants;

drop table if exists monitoring.heart_beats;

create table monitoring.heart_beats (
	heart_beat_id bigserial not null,
	instance_id int8 null,
	tenant_id int8 null,
	insert_date_time_utc timestamp(0) not null default timezone('utc'::text, now()),
	constraint heart_beats_pk primary key (heart_beat_id, insert_date_time_utc)
);

select create_hypertable('monitoring.heart_beats', 'insert_date_time_utc', chunk_time_interval => interval '12 hour');

create index on monitoring.heart_beats(insert_date_time_utc, instance_id) WITH (timescaledb.transaction_per_chunk);

drop table if exists tenants.tenants;
create table tenants.tenants (
	tenant_id bigserial,
	tenant_name character varying(255) not null,
	tenant_description character varying(500) null,
	insert_date_time_utc timestamp(0) null default timezone('utc'::text, now())
);

drop table if exists tenants.users;
create table tenants.users (
	user_id bigserial,
	tenant_id bigint not null,
	username character varying(255) not null,
	password character varying(255) not null,
	email character varying(255) null,
	first_name character varying(255) null,
	last_name character varying(255) null,
	insert_date_time_utc timestamp(0) NULL DEFAULT timezone('utc'::text, now())
);


drop table if exists tenants.tokens;
create table tenants.tokens (
	token_id bigserial not null,
	"token" character varying(500) not null,
	valid_until_utc timestamp without time zone,
	constraint tokens_pk primary key (token_id)
);

drop table if exists components.components;
create table components.components (
	component_id bigserial,
	component_name character varying(255) not null,
	component_description character varying(500) null,
	component_type smallint not null,
	tenant_id bigint not null,
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

create or replace function tenants.users_add(
	p_tenant_id bigint,
	p_username character varying(255),
	p_password character varying(255),
	p_email character varying(255),
	p_first_name character varying(255),
	p_last_name character varying(255))
 returns bigint
 language plpgsql
as $function$
	declare o_user_id bigint;
begin

	insert into tenants.users(tenant_id, username, password, email, first_name, last_name)
	values(p_tenant_id, p_username, tenants.crypt(p_password, tenants.gen_salt('bf', 8)), p_email, p_first_name, p_last_name)
	returning user_id into o_user_id;

	return o_user_id;

end;
$function$;

create or replace function tenants.users_authenticate(
	p_username character varying(255),
	p_password character varying(255))
 returns table(o_user_id bigint, o_email character varying(255), o_first_name character varying(255), o_last_name character varying(255))
 language plpgsql
as $function$
begin

	return query
	select user_id, email, first_name, last_name from tenants.users where username = p_username and "password" = tenants.crypt(p_password, "password");

end;
$function$;

drop function tenants.tokens_add(character varying,timestamp without time zone) ;
create or replace function tenants.tokens_add(p_token character varying(500), p_valid_until_utc timestamp without time zone)
 returns void
 language plpgsql
as $function$
begin

	insert into tenants.tokens("token", valid_until_utc)
	values(p_token, p_valid_until_utc);

end;
$function$;


create or replace function tenants.tokens_get_valid()
 returns table(o_token_id bigint, o_token character varying(500), o_valid_until timestamp without time zone)
 language plpgsql
as $function$
begin

	select token_id, "token", valid_until_utc 
	from tenants.tokens where valid_until_utc > now() at time zone 'utc';

end;
$function$;

```
