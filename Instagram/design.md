## System Design Probem - Instagram 

### Requirements and Goals of the System
Functional Requirements
1. Users should be able to upload/download/view photos.
2. Users can perform searches based on photo/video titles.
3. Users can follow other users.
4. The system should generate and display a user’s News Feed consisting of top photos from all the people the user follows.

Non-functional Requirements
1. Our service needs to be highly available.
2. The acceptable latency of the system is 200ms for News Feed generation.
3. Consistency can take a hit (in the interest of availability) if a user doesn’t see a photo for a while; it should be fine.
4. The system should be highly reliable; any uploaded photo or video should never be lost.

### Some Design Considerations
The system would be read-heavy
-  focus on building a system that can retrieve photos quickly.
1. users can upload as many photos as they like; therefore, efficient management of storage should be a crucial factor in designing this system.
2. Low latency is expected while viewing photos.
3. Data should be 100% reliable. If a user uploads a photo, the system will guarantee that it will never be lost.

### High Level System Design
- support two scenarios
    1. upload photos
    2. view/search photos
- object storage servers to store photos
- database servers to store metadata information about the photos.

### Database Schema
- Defining the DB schema in the early stages of the interview would help to understand the data flow among various components and later would guide towards data partitioning.
- relational databases come with their challenges, especially when we need to scale them.
- store photos in a distributed file storage like HDFS or S3.
- If we go with a NoSQL database, we need an additional table to store the relationships between users and photos to know who owns which photo.

### Data Size Estimation
- estimate how much data will be going into each table and how much total storage we will need for 10 years.
User:
- Assuming each “int” and “dateTime” is four bytes, each row in the User’s table will be of 68 bytes:
    - UserID (4 bytes) + Name (20 bytes) + Email (32 bytes) + DateOfBirth (4 bytes) + CreationDate (4 bytes) + LastLogin (4 bytes) = 68 bytes
- If we have 500 million users, we will need 32GB of total storage.
    - 500 million * 68 ~= 32GB
Photo: 
- row in Photo’s table will be of 284 bytes:
    - PhotoID (4 bytes) + UserID (4 bytes) + PhotoPath (256 bytes) + PhotoLatitude (4 bytes) + PhotoLongitude(4 bytes) + UserLatitude (4 bytes) + UserLongitude (4 bytes) + CreationDate (4 bytes) = 284 bytes
- If 2M new photos get uploaded every day, we will need 0.5GB of storage for one day:
    - 2M * 284 bytes ~= 0.5GB per day
    - For 10 years we will need 1.88TB of storage.
UserFollow: 
- Each row in the UserFollow table will consist of 8 bytes. If we have 500 million users and on average each user follows 500 users. We would need 1.82TB of storage for the UserFollow table:
    - 500 million users * 500 followers * 8 bytes ~= 1.82TB
- Total space required for all tables for 10 years will be 3.7TB:
    - 32GB + 1.88TB + 1.82TB ~= 3.7TB

### Component Design
- Uploading users can consume all the available connections, as uploading is a slow process. This means that ‘reads’ cannot be served if the system gets busy with all the ‘write’ requests. We should keep in mind that web servers have a connection limit before designing our system.
- we can split reads and writes into separate services. We will have dedicated servers for reads and different servers for writes to ensure that uploads don’t hog the system.

### Reliability and Redundancy
- Losing files is not an option for our service. Therefore, we will store multiple copies of each file so that if one storage server dies, we can retrieve the photo from the other copy present on a different storage server.