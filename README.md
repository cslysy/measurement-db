# measurement-db

## Schema overview

![schema-overview](/schema-overview.png)

## Tables

### device_type

Stores information about available devices type e.g. pressure gauge

```sql
CREATE TABLE device_type
(
  id serial NOT NULL,
  type character varying NOT NULL UNIQUE,
  CONSTRAINT device_type_pkey PRIMARY KEY (id)
);
```

### device
Stores information about concrete devices

```sql
CREATE TABLE device
(
  id serial NOT NULL,
  identifier character varying NOT NULL UNIQUE,
  device_type_id integer NOT NULL,
  CONSTRAINT device_pkey PRIMARY KEY (id),
  CONSTRAINT device_device_type_fkey FOREIGN KEY (device_type_id)
    REFERENCES device_type (id) MATCH SIMPLE
    ON UPDATE RESTRICT ON DELETE CASCADE
);
```

### measurement

Stores information about device measurement attempts

```sql
CREATE TABLE measurement
(
  id serial NOT NULL,
  date date NOT NULL,
  time time without time zone,
  device_id integer NOT NULL,
  CONSTRAINT measurement_pkey PRIMARY KEY (id),
  CONSTRAINT measurement_device_fkey FOREIGN KEY (device_id)
    REFERENCES device (id) MATCH SIMPLE
    ON UPDATE RESTRICT ON DELETE CASCADE,
  CONSTRAINT measurement_date_time_device_id_key UNIQUE (date, time, device_id)
);
```

### measurement_parameter

Stores set of parameters related to particular measurement

```sql
CREATE TABLE measurement_parameter
(
  id serial NOT NULL,
  value numeric(19, 9) NOT NULL,
  measurement_id integer NOT NULL,
  CONSTRAINT measurement_parameter_pkey PRIMARY KEY (id),
  CONSTRAINT measurement_parameter_measurement_fkey FOREIGN KEY (measurement_id)
    REFERENCES measurement (id) MATCH SIMPLE
    ON UPDATE RESTRICT ON DELETE CASCADE
);
```

## Indexes

Following indexes may be considered:

```sql
CREATE UNIQUE INDEX device_type_idx ON device_type (type);
CREATE UNIQUE INDEX device_identifier_idx ON device (identifier);
CREATE UNIQUE INDEX device_device_type_id_idx ON device (device_type_id);
CREATE UNIQUE INDEX measurement_date_idx ON measurement (date);
CREATE UNIQUE INDEX measurement_time_idx ON measurement (time);
CREATE UNIQUE INDEX measurement_device_id_idx ON measurement (device_id);
CREATE UNIQUE INDEX measurement_parameter_measurement_id_idx ON measurement_parameter (measurement_id);
```
As any report from reports below does not require matching measurement parameters value index for this column has not been created.

## Reports

Returns all `measurement_parameters` for the given `device` within defined time frame

```sql
SELECT device.identifier,
       measurement_parameter.value
FROM   device
       INNER JOIN measurement
               ON device.id = measurement.device_id
       INNER JOIN measurement_parameter
               ON measurement.id = measurement_parameter.measurement_id
WHERE  device.identifier = '212.53-T'
       AND measurement.date BETWEEN '2016-01-01' AND '2016-01-31'
```

Returns all `measurement_parameters` for all `devices` of the given `type` that took place on `2016-01-01`

```sql
SELECT measurement_parameter.value
FROM   measurement_parameter
       INNER JOIN measurement
               ON measurement_parameter.measurement_id = measurement.id
       INNER JOIN device
               ON measurement.device_id = device.id
       INNER JOIN device_type
               ON device.device_type_id = device_type.id
WHERE  device_type.type = 'pressure gauge'
       AND measurement.date = '2016-01-01'
```

Returns most recent `measurement_parameters` for all `devices` of the given `type` that took place on `2016-01-01`

```sql
SELECT m1.device_id,
       measurement_parameter.value AS measurement_parameter_value
FROM   measurement m1
       JOIN (SELECT device_id, max(time) AS recent_time
             FROM   measurement
             WHERE  date = '2016-01-01'
             GROUP  BY device_id
        ) m2 ON m1.device_id = m2.device_id AND m1.time = m2.recent_time
       INNER JOIN measurement_parameter ON measurement_parameter.measurement_id = m1.id
ORDER BY m1.device_id
```
