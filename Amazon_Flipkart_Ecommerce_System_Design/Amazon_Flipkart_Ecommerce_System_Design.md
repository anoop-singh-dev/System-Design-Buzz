# Briefing Document: E-Commerce System Design

## Introduction

This document summarizes the key architectural and design considerations for building a large-scale e-commerce platform as discussed in the provided source. The design emphasizes scalability, reliability, and a positive user experience, tackling the challenges of search, ordering, and personalization.

## I. Core Functional Requirements

The system must support the following core functions:

- **Product Search**: Users must be able to search for products and see delivery availability on the search results page itself. “We should also be able to tell them whether we can deliver it to them or not… at the page of search itself we should be able to tell them that we cannot deliver or if you are delivering then by when should we be able to deliver it to you.” This prioritizes a good user experience by avoiding the frustration of discovering undeliverable items late in the buying process.
- **Cart and Wishlist**: Users should be able to add items to carts and wishlists. These serve as intermediate steps before checkout and also provide data on user preferences.
- **Checkout and Payment**: Users must be able to complete purchases through a payment flow. While the details of payment gateway integration aren't covered, the broader flow is.
- **Order History**: Users must be able to view past order history including both delivered and undelivered items.

## II. Non-Functional Requirements (NFRs)

The system must adhere to the following NFRs:

- **Low Latency**: The system should be fast, as slow response times lead to poor user experience. “The system should have a very low latency. Why? - because it will be a bad user experience if it's slow.”
- **High Availability**: The system must be available for use as much as possible.
- **High Consistency**: Data must be reliable and accurate. However, the source acknowledges that not all components can meet all three NFRs simultaneously. Tradeoffs need to be made. “Not everything needs to have all these three. So some of the products which are mainly dealing with the payment and inventory counting, they need to be highly consistent at the cost of availability. Certain components, like search and all, they need to be highly available, maybe at the cost of consistent at times.”
- **Consistency vs Availability Tradeoffs**: Some services, like payment and inventory, need high consistency even if it impacts availability. User-facing components, such as search, need high availability, even at the cost of occasional inconsistency.
- **Low Latency**: Most user-facing components should strive for low latency.

## III. Architectural Overview

The system is composed of several layers:

- **User Interface (Green)**: Includes web browsers, mobile apps.
- **Load Balancers and Reverse Proxies**: These act as the entry point for requests and provide authentication/authorization.
- **Services (Blue)**: Various backend services handling different responsibilities, such as item management, search, order processing, etc.
- **Databases/Clusters (Red)**: Various databases and data stores, such as MongoDB, Elasticsearch, MySQL, Cassandra, Kafka, and Hadoop.

## IV. Key Services and Their Responsibilities

- **Inbound Service**: Manages integrations with various suppliers to acquire product data.
- **Item Service**: The source of truth for all items. It provides APIs for CRUD operations on items and also bulk retrieval. It uses MongoDB due to the flexible, non-structured nature of product attributes. "Item information is fundamentally very non-structured…that's the reason I use a Mongo database here."
- **Search Consumer**: Reads item data from Kafka and indexes it into Elasticsearch for fast text-based queries. It makes the product data searchable by the front end.
- **Search Service**: Provides APIs to search and filter products. It uses Elasticsearch for its text search and fuzzy search capabilities.
- **Serviceability and TAT (Turn Around Time) Service**: Determines delivery feasibility and estimated delivery times based on location, warehouse, and route capabilities. "…tries to figure out where exactly the product is. In what all warehouses... do I have a way to deliver products from this warehouse to the customer's pincode?"
- **User Service**: Manages user data, with a Redis cache for faster access to user information. “It’s a repository of all the users… It’ll sit on top of a MySQL database and a Redis Cache on top of it.”
- **Wishlist Service**: Manages user wishlists, storing data in MySQL.
- **Cart Service**: Manages user carts, storing data in MySQL.
- **Spark Streaming Consumer**: Analyzes user behavior (search, wishlist, cart) for reporting and recommendation generation.
- **Recommendation Service**: Stores product recommendations based on user behavior and ML models in a Cassandra database. It uses the data from the Spark consumer.
- **Logistic Service & Warehouse Service**: These two services are used by the Serviceability service, managing warehouse inventory details, routes, and delivery information. "For example, it might query this Warehouse Service to get a repository of all the items that are in the warehouse or it might query the Logistic Service to get details of all the pin codes that are existing or maybe details about the Courier Partners that work in a particular locality."
- **Order Taking Service**: Part of the Order Management System, responsible for creating orders and initiating the payment process. Uses MySQL for transactional consistency.
- **Inventory Service**: Manages product inventory levels. Crucial to ensure the correct product availability and handle race conditions. "... we want to block the inventory... we will reduce the inventory count at this point in time, and then send the user to payment."
- **Payment Service**: An abstraction over payment gateway integration. Handles payment confirmations and failures.
- **Reconciliation Service**: Checks that the inventory matches the orders that have been completed to identify issues or failed operations.
- **Archival Service**: Moves completed orders from MySQL to Cassandra for long-term storage.
- **Order Processing Service**: Handles the ongoing lifecycle of an order.
- **Historical Order Service**: Provides APIs for accessing archived order data in Cassandra.
- **Notification Service**: Manages notifications to customers/sellers about order status, etc.
- **Spark Jobs**: Run on top of Hadoop to perform various analytical functions to infer user behavior, reports, and recommendations.

## V. Data Flow Highlights

- **Data Ingestion**: Supplier data flows through the Inbound Service to Kafka.
- **Item Onboarding**: Item Service consumes new item data from Kafka, stores it in MongoDB, and provides APIs for item management.
- **Search Indexing**: Search Consumer reads item data from Kafka and indexes it into Elasticsearch.
- **Search Execution**: Search Service queries Elasticsearch for search results and integrates with Serviceability Service to filter out non-deliverable items. It also utilizes user data from the User Service.
- **Order Creation**: Order Taking Service creates order records in MySQL, blocks inventory, and initiates the payment flow. Redis is used to track order timeouts in case of payment failures.
- **Inventory Management**: Inventory Service manages stock levels, crucial to avoid over-selling products, as well as handle concurrent requests using constraints in the table schema.
- **Payment Processing**: Payment Service interacts with Payment Gateways, and sends events to the Order Taking Service for updating the order and inventory.
- **Data Analytics**: User actions and order data are sent to Kafka and consumed by Spark Streaming for real-time reports and batch processing for recommendation generation. This data is also stored in Hadoop for future analysis.
- **Order Archival**: Completed orders are moved from MySQL to Cassandra via the Archival Service.
- **Recommendations**: Recommendations are generated from Spark jobs and stored in Cassandra, and then presented to the user from the Recommendation service.

## VI. Database Choices

- **MongoDB**: Used for storing flexible, non-structured item data.
- **Elasticsearch**: Used for efficient full-text and fuzzy searches.
- **MySQL**: Used for transactional data where ACID properties are essential, such as for user data, cart, wishlist, and live order data.
- **Cassandra**: Used for long-term storage of historical orders and recommendations with specific query patterns.
- **Redis**: Used for caching user data and order state, specifically to handle timeouts.

## VII. Key Design Considerations

- **Event-Driven Architecture**: Kafka is heavily used for asynchronous communication between services.
- **Data Modeling**: Different types of data are stored in different databases based on data structure and access patterns.
- **Scalability and Performance**: The system is designed to scale using a distributed architecture.
- **Concurrency**: Inventory Service and the order management workflow are designed to handle concurrent requests and avoid over-selling.
- **Resiliency**: The system includes mechanisms to handle payment failures, timeouts, and data loss.
- **Optimization**: Pre-calculation of serviceability paths, caching user information, and efficient indexing improve overall system performance.

## VIII. Order Placement and Payment Flow

- **Atomic Updates**: MySQL is used to provide atomic transactions when orders are created and processed.
- **Inventory Blocking**: Inventory is blocked at the time the order is created to avoid over-selling.
- **Payment Confirmation**: If payment is successful, the order status is updated to PLACED.
- **Payment Failures**: If payment fails, the order is canceled and inventory is rolled back.
- **Timeouts**: Redis expiry is used to track payment timeouts and cancel orders in those scenarios.
- **Edge Cases**: The system needs to handle the potential risk conditions of overlapping payment success and expiry events.

## IX. Scalability Concerns

- **MySQL Bottleneck**: Acknowledge that the MySQL database for order data can become a bottleneck over time.
- **Order Archival**: Archival service is used to move completed orders to Cassandra for long-term storage to alleviate the problem.

## X. Future Considerations

- **Notification Service**: Detailed design is mentioned to be available in another video.
- **Machine Learning**: ML algorithms are used to build product recommendations.
- **User Similarity**: Recommendation engine uses user similarity in their recommendations.

## Conclusion

The provided source offers a comprehensive overview of the architecture of a large-scale e-commerce platform. It highlights key design choices, data flows, service responsibilities, and database usage, and most importantly addresses several key NFRs. The design focuses on achieving a balance between performance, scalability, consistency, and availability to provide a good user experience and a reliable platform. The use of various technologies and data stores is driven by specific requirements and tradeoffs. The document also touches upon edge cases, optimization techniques, and a means for future growth and analysis.
