---
layout: post
title: Using alternate login id without using alternate login id – part 2
date: 2018-05-23 07:10:25.000000000 +02:00
type: post
published: true
categories: 
    - ADFS
    - Authentication
tags:
    - migrated
    - popular 
permalink: "/2018/05/23/using-alternate-login-id-without-using-alternate-login-id-part-2/"
---

[In my first part](https://treeforestcloud.wordpress.com/2018/05/21/using-alternate-login-id-without-using-alternate-login-id-part-1/) I covered some of the backgrounds of this implementation and ended up with adding an additional SQL attribute store to ADFS. In this part I will explain setting up the database and tables for our alternative alternate login id.

The idea behind the alternative alternate login id is, to issue claims from a highly standardized data source (in our case a SQL database). So if you are dealing with a lot of Active Directory forests with a bunch of different UPN suffixes and want to have more control over the alternate login id process, this series is right for you. You can also take a look into the first part to learn some of the drawbacks in the out-of-box implementation of alternate login id.

Creating the database is straight forward. Of course you should align the settings with your dba. Because we will sync data into the database from Active Directory and ADFS will only read data out of it, backup is not so important to this database. **More important is a high availability of it - if the database is offline your claim rules flow will break and no authentication will happen!**

After creating the database you have to grant permissions to the ADFS farm. I suggest to grant access to the ADFS service account. If you want to use a SQL user you have to be sure to put username and password into the connection string in the attribute store. Regardless if you are using Windows integrated authentication or an SQL user the account needs the db_datareader role membership for the newly created database.

[gallery ids="67,68" type="rectangular"]

Now create a table to store all identities from the different Active Directory domains. You do not need all the attributes you have stored in Active Directory. I just use a few but it is up to you what you want to store. At least you need a primary key, the userPrincipalName and the attribute you want to use as your alternate login id. I will use the mail attribute. Here is a sample query I am also using in my environment.

```sql 
USE [SqlAttributeStore]  
GO

SET ANSI_NULLS ON  
GO

SET QUOTED_IDENTIFIER ON  
GO

CREATE TABLE [dbo].[Identities](  
[ObjectSID] [nvarchar](255) NOT NULL,  
[PUID] [nvarchar](12) NOT NULL,  
[LastName] [nvarchar](64) NOT NULL,  
[FirstName] [nvarchar](64) NOT NULL,  
[Displayname] [nvarchar](128) NOT NULL,  
[EmailAddress] [nvarchar](255) NULL,  
[SAMAccountName] [nvarchar](20) NULL,  
[UPN] [nvarchar](1024) NULL,  
[Company] [nvarchar](64) NOT NULL,  
[Domain] [nvarchar](255) NULL,  
[Country] [nvarchar](64) NOT NULL,  
[City] [nvarchar](max) NULL,  
CONSTRAINT [ObjectSID_PK] PRIMARY KEY CLUSTERED (  
[ObjectSID] ASC  
) WITH (  
PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON  
) ON [PRIMARY]  
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]  
GO  
```

#### Explanation for some of the columns

*   **ObjectSID** will be the primary key - this will contain the SID from the Active Directory object
*   **PUID** stands for person unique identifier and is a value I get out of our identity management system. It can also be the identifier from a human resource system.
*   **Domain** contains the NETBIOS domain name

The newly created table should be filled by a sync engine or a script on a regular base. One thing that was handy for me was to have a second table that is filled by hand. This is for example useful if you want to authenticate some accounts that are not in the scope of your sync jobs (an admin account for example)

So repeat the last step with a slightly different table name. I choosed [AdminIdentities] as the table name. Also the schema does not necessarily fully match your first table - I have removed the primary key SID because the table is filled only by hand and so I do not need an anchor.

```sql
USE [SqlAttributeStore]  
GO

SET QUOTED_IDENTIFIER ON  
GO

CREATE TABLE [dbo].[AdminIdentities] (  
[PUID] [nvarchar](12) NOT NULL,  
[LastName] [nvarchar](64) NOT NULL,  
[FirstName] [nvarchar](64) NOT NULL,  
[Displayname] [nvarchar](255) NOT NULL,  
[EmailAddress] [nvarchar](256) NOT NULL,  
[SAMAccountName] [nvarchar](20) NOT NULL,  
[UPN] [nvarchar](1024) NOT NULL,  
[Company] [nvarchar](64) NOT NULL,  
[Domain] [nvarchar](255) NOT NULL,  
[Country] [nvarchar](64) NOT NULL,  
[City] [nvarchar](64) NULL,  
[ID] [nvarchar](50) NULL  
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]  
GO  
```

Finally I wrap these two tables in a view. I strongly recommend that you use a view because you are more flexible when it comes to data querying, filtering and manipulation and the claim rules in ADFS stay very clean.  Here is an example of a view I am using:

```sql
USE [SqlAttributeStore]  
GO

SET ANSI_NULLS ON  
GO

SET QUOTED_IDENTIFIER ON  
GO

CREATE VIEW [dbo].[vPersons]  
AS  
SELECT dbo.Identities.PUID,  
dbo.Identities.LastName,  
dbo.Identities.FirstName,  
dbo.Identities.Displayname,  
dbo.Identities.EmailAddress,  
dbo.Identities.SAMAccountName  
dbo.Identities.UPN,  
dbo.Identities.Company,  
dbo.Identities.Domain,  
dbo.Identities.Country,  
dbo.Identities.City  
FROM dbo.Identities  
UNION  
SELECT dbo.AdminIdentities.PUID,  
dbo.AdminIdentities.LastName,  
dbo.AdminIdentities.FirstName,  
dbo.AdminIdentities.Displayname,  
dbo.AdminIdentities.EmailAddress,  
dbo.AdminIdentities.SAMAccountName,  
dbo.AdminIdentities.UPN,  
dbo.AdminIdentities.Company,  
dbo.AdminIdentities.Domain,  
dbo.AdminIdentities.Country,  
dbo.AdminIdentities.City,  
FROM dbo.AdminIdentities  
GO  
```

#### Example entries in the identity table

| ObjectSID | PUID | LastName | FirstName | Displayname | EmailAddress | UPN |
|----|----|----|----|----|----|----|
| 030300820-1013 | 51234 | Kline | Ann | Ann Kline | ann.kline@company.com | akline@domain.local |
| 030300820-4213 | 54256 | Johnson | Robert | Robert Johnson | robert.johnson@company.com | rjohnson@domain.local |
| 057560058-1272 | 54256 | Johnson | Robert | Robert Johnson | robert.johnson@company.com | rjoh@otherdomain.int |

As you can see I just added three data sets. I am focusing on the important columns. The SID is different for each entry. The first 2 are coming from the same domain (domain.local). The last entry belongs to the same real life person than the second one. This is why the PUID and email address is identically for both entries. This show very clear the beauty of this implementation. If you now use the PUID or at least the email adress in applications you can access them with different accounts from different domains and keep access permission and get a really amazing user experience.

#### Some more details about the PUID

I come back to the person unique identifier I mentioned above. If you have a chance to identify the real person by an id and don't have to rely only on technical account SIDs, you should absolutely add this id to the persons table.

I use ADFS for authentication for a lot of applications internally and always try to use the puid as the id in applications. Because doing so makes it much easier for administrators when users are moving between different domains or using multiple accounts. In the example above Robert Johnson could login to an so configured application with **rjoh@otherdomain.int **and **rjohnson@domain.local**

Because the application is setup up to use the PUID as user id it will not know that John is using different userPrincipalNames.  Of course you could also use the email address as the user id in an application but I do not recommend this. The email address can change over time and if this is your user id you will end up with user migrations in your application to keep access permissions and other internal application logic intact

#### Security concerns

When you are using the SQL attribute store keep in mind that you are now in the control of the authentication process. That means, everyone that has access to this database or to the ADFS farm, can modify the process and steal the identity of another user. I will come back to this when we implement the claim rule in the next part of this series and explain this threat a little bit more in detail.

#### Database Performance

If you are dealing with huge number of users and your identity table is getting quite big you should setup an index to some of your columns you are using to lookup the data. My observation is, that using a SQL database is very performant and can be optimized and different ways.

In the next part I will explain how to create claim rules for getting data out of the database and customize your claim rules.