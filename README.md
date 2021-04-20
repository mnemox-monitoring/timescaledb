create schema if not exists "monitoring";

create schema if not exists "components";

create schema if not exists "tenants";

ALTER USER postgres WITH PASSWORD '1qazxsw2';

create EXTENSION timescaledb UPDATE;

DROP TABLE monitoring.heart_beats;

CREATE TABLE monitoring.heart_beats (
	heart_beat_id bigserial,
	instance_id bigint NULL,
	insert_date_time_utc timestamp(0) NULL DEFAULT timezone('utc'::text, now())
);

SELECT create_hypertable('monitoring.heart_beats', 'insert_date_time_utc', chunk_time_interval => interval '12 hour');

CREATE INDEX ON monitoring.heart_beats(insert_date_time_utc, instance_id) WITH (timescaledb.transaction_per_chunk);

CREATE OR REPLACE FUNCTION monitoring.heart_beats_add(p_instance_id bigint)
 RETURNS void
 LANGUAGE plpgsql
AS $function$
begin

	insert into monitoring.heart_beats(instance_id)
	values(p_instance_id);

end;
$function$;


CREATE TABLE components.components (
	component_id bigserial,
	component_name character varying(255),
	component_description character varying(500),
	component_type smallint,
	insert_date_time_utc timestamp(0) NULL DEFAULT timezone('utc'::text, now())
);


CREATE OR REPLACE FUNCTION components.components_add(
	p_component_name character varying(255),
	p_component_description character varying(500),
	p_component_type smallint)
 RETURNS bigint
 LANGUAGE plpgsql
AS $function$
	declare o_component_id bigint;
begin

	insert into components.components(component_name, component_description, component_type)
	values(p_component_name, p_component_description, p_component_type)
	returning component_id into o_component_id;

end;
$function$;


CREATE TABLE tenants.tenants (
	tenant_id integer,
	tenant_name character varying(255),
	tenant_description character varying(500),
	insert_date_time_utc timestamp(0) NULL DEFAULT timezone('utc'::text, now())
);
