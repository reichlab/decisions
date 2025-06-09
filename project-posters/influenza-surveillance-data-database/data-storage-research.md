https://hevodata.com/learn/s3-vs-rds/

Feature	| Amazon RDS | Amazon S3
--------|---------------|---------------
1. Relational vs Object Storage | Relational storage with a fixed schema, suitable for structured data | Object storage for diverse data types (text, images, audio), without schema constraints
2. Support for Transactions | Supports multi-operation transactions with consistency and rollback | Strong consistency for single operations; lacks built-in transaction support
3. Data Processing | Integrated processing engine supports complex queries and aggregations | Requires external tools (e.g., Athena, Redshift Spectrum) for processing and querying data
4. Pricing | Higher costs; varies by database engine and instance type | Lower cost for storage; pay-as-you-go for additional requests and data transfer
5. Use Cases | Ideal for structured data with frequent updates, transactional workloads, or user applications | Suitable for unstructured data, media files, backups, and data lake or staging area for ETL


https://databasetown.com/relational-database-vs-object-oriented-database-key-differences/

Criteria | Relational Database | Object Oriented Database
--------|---------------|---------------|---------------
Definition | Data is stored in tables which consist of rows and columns. | Data is stored in objects. Objects contain data.
Amount of data | It can handle large amounts of data. | It can handle larger and complex data.
Type of data | Relational database has single type of data. | It can handle different types of data.
How data is stored | Data is stored in the form of tables (having rows and columns). | Data is stored in the form of objects.
Data Manipulation Language | DML is as powerful as relational algebra. Such as SQL, QUEL and QBE. | DML is incorporated into object-oriented programming languages, such as C++, C#.  
Learning | Learning relational database is a bit complex. | Object oriented databases are easier to learn as compared to relational database.
Structure | It does not provide a persistent storage structure because all relations are implemented as separate files.	| It provides persistent storage for objects (having complex structure) as it uses indexing technique to find the pages that store the object.
Constraints | Relational model has key constraints, domain constraints, referential integrity and entity integrity constraints. | To check the integrity constraints is a basic problem in object-oriented database.
Cost | The maintenance cost of relational database may be lower than the cost of expertise required development and integration of object oriented database. | In some cases hardware and software cost of object oriented databases is lower cost than relational databases.


https://www.ibm.com/think/topics/data-warehouse-vs-data-lake-vs-data-lakehouse

Feature | Data Warehouses | Data Lakes | Data Lakehouses
--------|---------------|---------------|---------------
Types of data | Mostly structured | Raw structured, unstructured, semi-structured | Raw or processed structured, unstructured and semi-structured
Storage type | Relational data bases | Cloud object storage | Cloud objects storage
Storage cost | High | Low | Low
Native query performance | High | Low | High
Schema approach | Schema on right | Schema on read | Supports both
Defining feature | Built-in analytics engine supports high performance | Low cost, scalable storage for all data types | Warehouse performance and lake storage in one solution
Common use cases | Business intelligence and data analytics | AI, machine learning, and data science | All of the above

---

Note that a database will require a server while something like an S3 bucket will not
