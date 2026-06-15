Step-by-Step Local Setup
────────────────────────

Step 1 — Prerequisites
bash ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
# Java 11+
java -version   # should be 11+

# Maven
mvn -version

# Install Cassandra via brew
brew install cassandra                                                                                                                  
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Step 2 — Find your local datacenter name ⚠️
This is the most common gotcha. Brew-installed Cassandra uses datacenter1 not dc1:
bash ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
# Start Cassandra
brew services start cassandra

# Wait ~30s then check your DC name
cqlsh -e "SELECT data_center FROM system.local;"
# You'll see either 'datacenter1' or 'dc1'
# Use THAT value in the application.yml changes above
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Step 3 — Create the keyspace and schema
bash ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
cqlsh << 'EOF'                                                                                                                          
CREATE KEYSPACE IF NOT EXISTS betterbotz                                                                                                
WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};

USE betterbotz;

-- products table (spring-boot service manages this via ProductDao)                                                                     
CREATE TABLE IF NOT EXISTS products (                                                                                                   
name text,                                                                                                                            
id uuid,                                                                                                                              
description text,                                                                                                                     
price decimal,                                                                                                                        
PRIMARY KEY ((name), id)                                                                                                              
);

-- orders table (from microservice-spring-data/src/main/resources/orders-schema.cql)                                                    
CREATE TABLE IF NOT EXISTS orders (                                                                                                     
order_id uuid,                                                                                                                        
product_id uuid,                                                                                                                      
product_name text,                                                                                                                    
product_quantity int,                                                                                                                 
product_price decimal,                                                                                                                
added_to_order_at timestamp,                                                                                                          
PRIMARY KEY ((order_id), product_id)                                                                                                  
);                                                                                                                                      
EOF                                                                                                                                     
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Step 4 — Clone and apply the code changes above
bash ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
git clone https://github.com/sathishragu-gh/spring-k8s-cassandra-microservices.git                                                    
cd spring-k8s-cassandra-microservices
# Apply all 5 changes above
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Step 5 — Build
bash ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
# Skip tests (no test infra needed)
mvn clean package -DskipTests                                                                                                           
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Step 6 — Start the 3 services (3 separate terminals)

Terminal 1 — Spring Boot service (products):
bash ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
cd microservice-spring-boot                                                                                                             
java -jar target/microservice-spring-boot-1.0-SNAPSHOT.jar
# Starts on :8083
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Terminal 2 — Spring Data service (orders):
bash ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
cd microservice-spring-data                                                                                                             
java -jar target/microservice-spring-data-1.0-SNAPSHOT.jar
# Starts on :8081
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Terminal 3 — Gateway:
bash ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
cd gateway-service                                                                                                                      
java -jar target/gateway-service-1.0-SNAPSHOT.jar
# Starts on :8080
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Step 7 — Test it
bash ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
# Add a product (direct to spring-boot service)
curl -X POST -H "Content-Type: application/json" \                                                                                      
-d '{"name":"mobile","id":"123e4567-e89b-12d3-a456-556642440000","description":"iPhone","price":"500.00"}' \                          
http://localhost:8083/api/products/add

# Query via gateway (routes through :8080 → :8083)
curl http://localhost:8080/api/products/search/mobile

# Add an order (direct to spring-data service)
curl -H "Content-Type: application/json" \                                                                                              
-d '{"key":{"orderId":"123e4567-e89b-12d3-a456-556642440000","productId":"123e4567-e89b-12d3-a456-556642440000"},"productName":"iPhone","productPrice":"500.00","productQuantity":1,"addedToOrderTimestamp":"2021-01-01T00:00:00Z"}' \
http://localhost:8081/api/orders

# Swagger UI
open http://localhost:8083/swagger-ui.html                                                                                              
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

───