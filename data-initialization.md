```sql

delete from tenants.users;
delete from tenants.tokens;
delete from tenants.tenants;

insert into tenants.tenants(tenant_name, tenant_description)
values('default tenant', 'default tenant');

select tenants.users_add((select tenant_id from tenants.tenants limit 1), 
	'mnemox'::character varying, 
	'mnemox'::character varying, 
	null::character varying, 
	null::character varying, 
	null::character varying);

```
