# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This is the official repository for the Rock the JVM Spark Essentials with Scala course. The codebase is structured as a learning progression from basic Scala concepts through advanced Spark applications, with both local development and distributed cluster deployment capabilities.

## Development Environment Setup

### Prerequisites
- Install [Docker](https://docker.com) and Docker Compose
- Java Development Kit (JDK) 11 or higher
- SBT (Scala Build Tool)
- IntelliJ IDEA (recommended)

### Initial Setup Commands

```bash
# Clone and open as SBT project in IntelliJ
# Import as existing SBT project

# Download SBT dependencies
sbt update

# Start PostgreSQL database (from repo root)
docker compose up

# Build Spark cluster Docker images (from spark-cluster/ directory)
cd spark-cluster
chmod +x build-images.sh
./build-images.sh
```

**Windows Users**: If you encounter `'\r': command not found` errors, run `dos2unix` on all `.sh` files in the spark-cluster directory before building images.

### Start Development Environment

```bash
# Start PostgreSQL (repo root)
docker compose up

# Start Spark cluster with 3 workers (from spark-cluster/)
cd spark-cluster
docker compose up --scale spark-worker=3
```

## Core Build and Development Commands

### SBT Operations
```bash
# Compile the project
sbt compile

# Run tests (if any exist)
sbt test

# Reload dependencies after build.sbt changes
sbt reload

# Start Scala REPL with project dependencies
sbt console
```

### Running Applications
```bash
# Run individual lessons/applications
sbt "runMain part2dataframes.DataFramesBasics"
sbt "runMain part3typesdatasets.Datasets"
sbt "runMain part6practical.TestDeployApp"
sbt "runMain part7bigdata.TaxiApplication"

# Run playground for experimentation
sbt "runMain playground.Playground"
```

### Building JAR for Cluster Deployment
The project uses IntelliJ's artifact system for creating deployment JARs:
1. Project Structure → Artifacts → Add → JAR → From modules with dependencies
2. Select main class (e.g., `part6practical.TestDeployApp`)
3. Check "copy to the output folder and link to manifest"
4. Build → Build Artifacts → select the jar → build

## Docker Infrastructure Commands

### PostgreSQL Database
```bash
# Start database
docker compose up

# Connect to PostgreSQL container
docker exec -it postgres psql -U docker -d rtjvm

# Stop database
docker compose down
```

### Spark Cluster Management
```bash
# Build Spark cluster images (one-time setup)
cd spark-cluster
./build-images.sh  # or build-images.bat on Windows

# Start cluster with 3 workers
docker compose up --scale spark-worker=3

# Stop cluster
docker compose down

# Access Spark Master UI
# http://localhost:9090 (Spark Master UI)
# http://localhost:4040 (Spark Application UI when running)

# Connect to master container for spark-submit
docker exec -it spark-cluster_spark-master_1 bash
```

### Deploying Applications to Cluster
```bash
# Copy JAR and data files to cluster (from repo root)
cp target/scala-2.13/spark-essentials.jar spark-cluster/apps/
cp src/main/resources/data/movies.json spark-cluster/data/

# Submit job to cluster (inside master container)
/spark/bin/spark-submit \
  --class part6practical.TestDeployApp \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --verbose \
  --supervise \
  /opt/spark-apps/spark-essentials.jar \
  /opt/spark-data/movies.json \
  /opt/spark-data/goodMovies
```

## Architecture Overview

### Code Organization
```
src/main/scala/
├── part1recap/           # Scala fundamentals recap
├── part2dataframes/      # DataFrame operations (basics, joins, aggregations)
├── part3typesdatasets/   # Spark types, Datasets, null handling
├── part4sql/            # Spark SQL, database connectivity
├── part5lowlevel/       # RDD operations
├── part6practical/      # Deployment and practical applications
├── part7bigdata/        # Large dataset processing (taxi data analysis)
└── playground/          # Experimentation and testing

src/main/resources/data/  # Sample datasets (JSON, CSV, Parquet)
spark-cluster/           # Docker-based Spark cluster setup
sql/                    # PostgreSQL initialization scripts
```

### Development Flow
```
Local Development (spark.master = "local")
    ↓
Build JAR (IntelliJ Artifacts or sbt assembly)
    ↓
Copy to spark-cluster/apps/
    ↓
Submit to Docker Cluster (spark://spark-master:7077)
```

### Key Dependencies (build.sbt)
- Scala 2.13.12
- Spark 3.5.0 (Core + SQL)
- PostgreSQL Driver 42.6.0
- Log4j 2.20.0

## Key Development Patterns

### SparkSession Initialization
```scala path=null start=null
val spark = SparkSession.builder()
  .appName("Your Application Name")
  .config("spark.master", "local")  // For local development
  .getOrCreate()
```

### Reading Data Sources
```scala path=null start=null
// JSON with schema inference
val df = spark.read
  .format("json")
  .option("inferSchema", "true")
  .load("src/main/resources/data/cars.json")

// CSV with headers
val csvDF = spark.read
  .option("header", "true")
  .option("inferSchema", "true")
  .csv("src/main/resources/data/file.csv")

// PostgreSQL table
val dbDF = spark.read
  .format("jdbc")
  .option("driver", "org.postgresql.Driver")
  .option("url", "jdbc:postgresql://localhost:5432/rtjvm")
  .option("user", "docker")
  .option("password", "docker")
  .option("dbtable", "public.tablename")
  .load()
```

### DataFrame Operations Pattern
```scala path=null start=null
import spark.implicits._
import org.apache.spark.sql.functions._

val result = df
  .select(col("column1"), col("column2").as("renamed"))
  .where(col("column1") > 100)
  .groupBy("column2")
  .agg(count("*").as("count"), avg("column1").as("average"))
  .orderBy(col("count").desc)
```

### Writing Results
```scala path=null start=null
df.write
  .mode(SaveMode.Overwrite)
  .format("json")
  .save("output/path")
```

## Testing and Validation

### Local Testing
```bash
# Test basic functionality
sbt "runMain playground.Playground"

# Test DataFrame operations
sbt "runMain part2dataframes.DataFramesBasics"

# Test database connectivity (requires running PostgreSQL)
sbt "runMain part4sql.SparkSql"
```

### Cluster Deployment Testing
1. Build JAR using IntelliJ artifacts
2. Copy JAR to `spark-cluster/apps/`
3. Copy test data to `spark-cluster/data/`
4. Start cluster: `cd spark-cluster && docker compose up --scale spark-worker=3`
5. Submit job:
   ```bash
   docker exec -it spark-cluster_spark-master_1 bash
   /spark/bin/spark-submit --class part6practical.TestDeployApp --master spark://spark-master:7077 /opt/spark-apps/spark-essentials.jar /opt/spark-data/movies.json /opt/spark-data/output
   ```
6. Monitor job progress at http://localhost:9090

### Cleanup
```bash
# Stop all containers
docker compose down

# Clean cluster
cd spark-cluster
docker compose down

# Remove volumes if needed
docker volume prune
```

## Branch Navigation

- `start`: Clean starting point for following the course
- `master`: Complete code for Rock the JVM students  
- `udemy`: Complete code for Udemy students

```bash
# Switch to starting point
git checkout start

# View completed solutions
git checkout master    # or 'udemy' for Udemy version
```

## Important Notes

- All applications default to `spark.master = "local"` for development
- PostgreSQL runs on localhost:5432 with credentials `docker/docker`
- Spark Master UI available at http://localhost:9090 when cluster is running
- Sample datasets are pre-loaded in `src/main/resources/data/`
- Windows users need Hadoop native binaries (see HadoopWindowsUserSetup.md)
- Volume mounts: `spark-cluster/apps` → `/opt/spark-apps`, `spark-cluster/data` → `/opt/spark-data`
