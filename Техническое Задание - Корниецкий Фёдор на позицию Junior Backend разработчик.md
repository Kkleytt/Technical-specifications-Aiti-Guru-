## Создание таблиц
#### Даталогическая модель

<img width="2808" height="2328" alt="изображение" src="https://github.com/user-attachments/assets/3c488f09-f463-45a0-8e97-1b9a27f343da" />

#### Таблица номенклатуры

| Поле          | Тип данных                                               | Назначение и обоснование выбора                                                                                                                                                                                                                                                                                               |
| ------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **id**        | `SERIAL PRIMARY KEY`                                     | Уникальный идентификатор категории. `SERIAL` автоматически увеличивает значение при добавлении новой записи, что удобно для первичного ключа.                                                                                                                                                                                 |
| **name**      | `VARCHAR(255) NOT NULL`                                  | Название категории. `VARCHAR(255)` позволяет хранить названия до 255 символов, чего достаточно для большинства случаев. `NOT NULL` гарантирует, что каждая категория будет иметь имя.                                                                                                                                         |
| **parent_id** | `INTEGER REFERENCES nomenclature(id) ON DELETE SET NULL` | Ссылка на родительскую категорию. `INTEGER` — стандартный тип для хранения внешнего ключа (id). `REFERENCES nomenclature(id)` реализует связь с самой собой, позволяя строить дерево категорий. `ON DELETE SET NULL` обеспечивает сохранность дочерних категорий при удалении родительской — они просто становятся корневыми. |

``` SQL

CREATE TABLE nomenclature (
	id SERIAL PRIMARY KEY,
	name VARCHAR(255) NOT NULL,
	parent_id INTEGER REFERENCES nomenclature(id) ON DELETE SET NULL
);
```
  
#### Таблица каталога номенклатуры

| Поле              | Тип данных                                               | Назначение и обоснование выбора                                                                |
| ----------------- | -------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `id`              | `SERIAL PRIMARY KEY`                                     | Уникальный ID товара. `SERIAL` — автоинкремент.                                                |
| `name`            | `VARCHAR(255) NOT NULL`                                  | Название товара. `VARCHAR(255)` — стандарт для текстовых данных.                               |
| `quantity`        | `INTEGER NOT NULL`                                       | Количество товара. `INTEGER` — целочисленный тип, `NOT NULL` исключает пустые значения.        |
| `price`           | `DECIMAL(10,2) NOT NULL`                                 | Цена товара. `DECIMAL(10,2)` — точный тип для денежных значений с двумя знаками после запятой. |
| `nomenclature_id` | `INTEGER REFERENCES nomenclature(id) ON DELETE SET NULL` | Ссылка на категорию. `ON DELETE SET NULL` — товар остаётся, даже если категория удалена.       |

``` SQL
CREATE TABLE catalog_nomenclature (
	id SERIAL PRIMARY KEY,
	name VARCHAR(255) NOT NULL,
	quantity INTEGER NOT NULL,
	price DECIMAL(10,2) NOT NULL,
	nomenclature_id INTEGER REFERENCES nomenclature(id) ON DELETE SET NULL
);
```


#### Таблица клиентов

| Поле      | Тип данных              | Назначение и обоснование выбора                                          |
| --------- | ----------------------- | ------------------------------------------------------------------------ |
| `id`      | `SERIAL PRIMARY KEY`    | Уникальный ID клиента. `SERIAL` — автоинкремент.                         |
| `name`    | `VARCHAR(255) NOT NULL` | Имя клиента. `VARCHAR(255)` — стандарт для текстовых данных.             |
| `address` | `TEXT`                  | Адрес клиента. `TEXT` — гибкий тип, подходит для хранения длинных строк. |

``` SQL
CREATE TABLE client (
	id SERIAL PRIMARY KEY,
	name VARCHAR(255) NOT NULL,
	address TEXT
);
```


#### Таблица заказов

|Поле|Тип данных|Назначение и обоснование выбора|
|---|---|---|
|`id`|`SERIAL PRIMARY KEY`|Уникальный ID заказа. `SERIAL` — автоинкремент.|
|`client_id`|`INTEGER REFERENCES client(id) ON DELETE CASCADE`|Ссылка на клиента. `ON DELETE CASCADE` — удаление клиента удаляет его заказы.|
|`created_at`|`TIMESTAMP DEFAULT CURRENT_TIMESTAMP`|Дата создания заказа. `TIMESTAMP` — точный тип для хранения даты и времени.|

``` SQL
CREATE TABLE "order" ( /*Заключено в ковычки так как order - зарезервированное имя в SQL*/
	id SERIAL PRIMARY KEY,
	client_id INTEGER REFERENCES client(id) ON DELETE CASCADE,
	created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Таблица состава заказа

| Поле       | Тип данных                                         | Назначение и обоснование выбора                                                |
| ---------- | -------------------------------------------------- | ------------------------------------------------------------------------------ |
| `id`       | `SERIAL PRIMARY KEY`                               | Уникальный ID позиции заказа. `SERIAL` — автоинкремент.                        |
| `order_id` | `INTEGER REFERENCES "order"(id) ON DELETE CASCADE` | Ссылка на заказ. `ON DELETE CASCADE` — удаление заказа удаляет его позиции.    |
| `catalog_nomenclature_id`  | `INTEGER REFERENCES catalog_nomenclature(id) ON DELETE CASCADE`    | Ссылка на товар. `ON DELETE CASCADE` — удаление товара удаляет его из заказов. |
| `quantity` | `INTEGER NOT NULL`                                 | Количество единиц товара в заказе. `INTEGER` — целочисленный тип.              |

``` SQL
CREATE TABLE order_item (
	id SERIAL PRIMARY KEY,
	order_id INTEGER REFERENCES "order"(id) ON DELETE CASCADE,
	catalog_nomenclature_id INTEGER REFERENCES catalog_nomenclature(id) ON DELETE CASCADE,
	quantity INTEGER NOT NULL
);
```

## SQL запросы

#### Сумма купленных товаров для каждого клиента

``` SQL
SELECT 
    c.name AS client_name,
    SUM(oi.quantity * n.price) AS total_order_amount
FROM 
    client c
JOIN 
    "order" o ON c.id = o.client_id
JOIN 
    order_item oi ON o.id = oi.order_id
JOIN 
    catalog_nomenclature n ON oi.catalog_nomenclature_id = n.id
GROUP BY
	c.name;
```

#### Число дочерних товаров

``` SQL
SELECT 
    parent.name AS category_name,
    COUNT(child.id) AS first_level_children_count
FROM 
    nomenclature parent
LEFT JOIN 
    nomenclature child ON child.parent_id = parent.id
GROUP BY 
    parent.id, parent.name
ORDER BY 
    parent.name;
```
