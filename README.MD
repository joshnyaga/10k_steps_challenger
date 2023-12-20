4 public services
===================
2 user facing web services
1 for receiving pedometer updates
1 to expose a public HTTPS

4 internal services
============================
1 to mange user profiles
1 to manage activity data
1 congratulate the user
1 to compute various stats over continous events


How it works
=================================
It starts with the pedometer sending an update to the ingestion service. Ingestion service sends an update to a kafka topic. The
pedometer then acknoledges that an update has been received and will process it. The update has multiple consumers listening to 
that particular kafka topic. One of the consumers is the activity service which then sends the update to Postgress database. The service will then record another topic with number of steps recorded by the pedometer on that day. This event will be picked by the events service which picks updates after every 5 seconds, splits them in to city and aggregates the number of steps. The event service then posts an update as a kafka topic with the increment steps observed for a given city. This record is consumed by  the dashboard web application which finally sends an update to all connected web browsers then updates the display.



1. USER PROFILE SERVICE
=================================
user is identified by:
1. A username(must be unique)
2. A password
3. An email address
4. A city
5. A pedometer device identifier(Must be unique)
6. whether the user wants to appear in public ranking or not


The service used Mongo DB data persistence. It has CRUD. It is not publicly exposed meaning it is consumed by other services.

endpoints
Purpose         Path                Method      Data        Response        Status Code
--------------------------------------------------------------------------------------
Register 
new user        /register           POST        JSON        N/A             200, 409, 500, 422

Get User        /<username>         GET         N/A         user data-JSON  200, 404, 500

Update user
details         /<username>         PUT         JSON        N/A             200, 500, 422, 409

Credentials
validation/
login           /authenticate       POST        JSON        N/A             200, 401, 422    

Reverse lookup
of a user by 
device          /owns/<deviceId>    GET         N/A         JSON data       200, 404, 500



2. Ingestion Service
========================================
collects pedometer device, updates and forwards records with update data to a kafka stream for other services to process the event. It receives update via HTTP API or AMQP Queue. It is a form of protocol adapter/mediator coz it is converting HTTP/AMQP to KAFKA Record Streams.
A device update is a JSON doc with the following entries
1. device identifier
2. synchronization identifier increasing long integer 
3 The number of steps since last synchronization

Purpose         Path                Method      Data        Response        Status Code
--------------------------------------------------------------------------------------
ingest a 
pedometer
update          /ingest             POST        JSON Doc    N/A             200, 500, 422


The service does not check for duplicates.


3. Activity Service
==========================================
Stores events to a PostgreSQL db and offers an HTTP API to gather some statistics like monthly and total steps

Purpose         Path                                Method      Data        Response        Status Code
-------------------------------------------------------------------------------------------
Total Steps     /<device id>/total                  GET         N/A         Json doc        200, 404, 500

Monthly step
count           /<device-id>/<year>/<month>         GET         N/A         Json            200,404,500

daily step 
count           /device-id>/<year>/<month>/<day>    GET         N/A         Json            200,404,500

Device Ranking
steps over last 
 24 hours       /ranking-last-24-hours              GET         N/A         Json            200,500


Events sent to daily step should contain a JSON Object with the following
1. The device identifier
2. A timestamp
3. Number of steps receorded in the current day

For each incoming update, the following operations will be performed
1. database insert
2. Query to get number of steps
3. kafka record write


4. PUBlIC API
=============================================================================
Exposes a public HTTP API and other services to consume, Acts as an api gateway and composes requests to other services. It requires authentication for most operations. We use JSON Web tokens for this. We need to enable CORS too for this service coz of accessing it via the web



Purpose         Path                                Method      Data        Response        Status Code
----------------------------------------------------------------------------------------
Register 
new user        /register                           POST        JSON        N/A             200, 502, 422


Get a JWT
Toke            /token                              POST        JSON        JWT Token       200, 401, 422

Get user data   /<username>                         GET         N/A         Json            200, 404,502

Update user
details         /<username>                         PUT         JSON        N/A             200, 404, 502, 422

total steps     /<username>/total                   GET         N/A         JSON            200, 502, 404

Monthly step
count           /<device-id>/<year>/<month>         GET         N/A         Json            200,404,502

daily step 
count           /device-id>/<year>/<month>/<day>    GET         N/A         Json            200,404,502

