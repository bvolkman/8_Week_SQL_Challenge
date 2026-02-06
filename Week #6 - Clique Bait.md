# Case Study #6 - Clique Bait

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/27734bf7-3f04-4650-9eab-3483a6af32b9" />

## Summary

Clique Bait is not like your regular online seafood store - the founder and CEO Danny, was also a part of a digital data analytics team and wanted to expand his knowledge into the seafood industry!

In this case study - you are required to support Danny’s vision and analyse his dataset and come up with creative solutions to calculate funnel fallout rates for the Clique Bait online store.

**Table 1: Users**

Customers who visit the Clique Bait website are tagged via their `cookie_id`.

**Table 2: Events**

Customer visits are logged in this `events` table at a `cookie_id` level and the `event_type` and `page_id` values can be used to join onto relevant satellite tables to obtain further information about each event. The `sequence_number` is used to order the events within each visit.

**Table 3: Event Identifier**

The `event_identifier` table shows the types of events which are captured by Clique Bait’s digital data systems.

**Table 4: Campaign Identifier**

This table shows information for the 3 campaigns that Clique Bait has ran on their website so far in 2020.

**Table 5: Page Hierarchy**

This table lists all of the pages on the Clique Bait website which are tagged and have data passing through from user interaction events.

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/jmnwogTsUE8hGqkZv9H7E8/17). Follow the link to view the schema and feel free to copy and paste the codes to see them in action.

### 1. Enterprise Relationship Diagram

Create an ERD for all the Clique Bait datasets using this link https://dbdiagram.io/d.
````
Table event_identifier {	
event_type integer	
event_name varchar(13)	
}	
	
Table campaign_identifier {	
campaign_id integer [primary key]	
products varchar(3)	
campaign_name varchar(33)	
start_date timestamp	
end_date timestamp	
}	
	
Table page_hierarchy {	
page_id integer [primary key]	
page_name varchar(14)	
product_category varchar(9)	
product_id integer	
}	
	
Table users {	
user_id integer	
cookie_id varchar(6)	
start_date timestamp	
}	
	
Table events {	
visit_id varchar(6)	
cookie_id varchar(6)	
page_id integer	
event_type integer	
sequence_number integer	
event_time timestamp	
}	
	
Ref: users.cookie_id > events.cookie_id	
	
Ref: events.page_id > page_hierarchy.page_id	
	
Ref: events.event_type > event_identifier.event_type
````
Result:
<img width="1806" height="926" alt="image" src="https://github.com/user-attachments/assets/b67b3315-260c-43c5-861c-668be30fa4ba" />
#

### 2. Digital Analysis
