时间戳在存储时主要有两种格式: 1. 带有时区信息 2. 不带时区信息.

在 postgreSQL 数据库中, `timestamp` 类型表示不带时区的时间戳. 而 `timestamptz` 表示带有时区数据的时间戳.

在 ecto 中, 可以按以下方式选择要使用的时间戳类型:

在 migration 文件中:
```ex
create table(:events) do

  timestamps(type: :timestamptz)
end
```

在 schema 文件中:

```ex
schema "events" do

  timestamps(type: :utc_datetime)
end
```

使用 `:utc_datetime` 会让你在读取数据的时候始终获得UTC 时区的时间戳.


