# Postgres logs using trigger

###Table to save logs
```sql
CREATE TABLE clientes (
    id serial NOT NULL
        CONSTRAINT clientes_pk
            PRIMARY KEY,
    name varchar(200) NOT NULL,
    data_nascimento date,
    cpf varchar(14) NOT NULL,
    renda_mensal numeric(12, 2),
    ativo boolean DEFAULT TRUE NOT NULL
);
```

###Log Table
```sql
CREATE TABLE logs (
    id serial NOT NULL
        CONSTRAINT logs_pk
            PRIMARY KEY,
    tablename varchar(255) NOT NULL,
    old jsonb,
    new jsonb,
    date timestamp DEFAULT NOW() NOT NULL,
    query text NOT NULL,
    action char NOT NULL CHECK (action IN ('I','D','U')),
    schema varchar(255) NOT NULL,
    database_user varchar(255) NOT NULL
);
```

###Trigger
```sql
CREATE OR REPLACE function public.log() returns trigger
    language plpgsql
as
$$
DECLARE
    old_data jsonb;
    new_data jsonb;
BEGIN
    IF (TG_OP = 'UPDATE') THEN
        IF OLD is distinct from NEW THEN
            old_data = row_to_json(OLD);
            new_data = row_to_json(NEW);

            INSERT INTO public.logs (schema,tablename,database_user,action,old,new,query)
            VALUES (
                    TG_TABLE_SCHEMA::TEXT,
                    TG_TABLE_NAME::TEXT,
                    session_user::TEXT,
                    'U',
                    old_data,
                    new_data,
                    current_query()
            );
        END IF;

        RETURN NEW;
    ELSIF (TG_OP = 'DELETE') THEN
        old_data = row_to_json(OLD);

        INSERT INTO public.logs (schema,tablename,database_user,action,old,query)
        VALUES (
                TG_TABLE_SCHEMA::TEXT,
                TG_TABLE_NAME::TEXT,
                session_user::TEXT,
                'D',
                old_data,
                current_query()
        );
        RETURN OLD;
    ELSIF (TG_OP = 'INSERT') THEN
        new_data = row_to_json(NEW);
        INSERT INTO public.logs (schema,tablename,database_user,action,new,query)
        VALUES (
                TG_TABLE_SCHEMA::TEXT,
                TG_TABLE_NAME::TEXT,
                session_user::TEXT,
                'I',
                new_data,
                current_query()
        );
        RETURN NEW;
    END IF;
END;
$$;
```

###Define trigger in table
```sql
CREATE TRIGGER trigge_log
    AFTER INSERT OR UPDATE OR DELETE
    ON clientes
    FOR EACH ROW
EXECUTE PROCEDURE log();
```