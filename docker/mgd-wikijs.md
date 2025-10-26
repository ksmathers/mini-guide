# WikiJS Docker Container Setup (Mini Guide)

## ✅ Part 1: Prerequisites and Setup

### 1. Ensure Docker is installed and running
```bash
docker --version
docker info
```

### 2. Create directories for persistent storage
```bash
mkdir -p ~/wikijs/{data,config}
cd ~/wikijs
```

## ✅ Part 2: Database Setup (PostgreSQL)

### 3. Create a Docker network for WikiJS components
```bash
docker network create wikijs-network
```

### 4. Start PostgreSQL database container
```bash
docker run -d \
  --name wikijs-db \
  --network wikijs-network \
  -e POSTGRES_DB=wiki \
  -e POSTGRES_USER=wikijs \
  -e POSTGRES_PASSWORD=wikijspass \
  -v ~/wikijs/data/postgres:/var/lib/postgresql/data \
  postgres:15-alpine
```

### 5. Verify database is running
```bash
docker ps | grep wikijs-db
docker logs wikijs-db
```

## ✅ Part 3: WikiJS Container Setup

### 6. Start WikiJS container
```bash
docker run -d \
  --name wikijs \
  --network wikijs-network \
  -p 3000:3000 \
  -e DB_TYPE=postgres \
  -e DB_HOST=wikijs-db \
  -e DB_PORT=5432 \
  -e DB_USER=wikijs \
  -e DB_PASS=wikijspass \
  -e DB_NAME=wiki \
  -v ~/wikijs/config:/wiki/config \
  requarks/wiki:2
```

### 7. Verify WikiJS container is running
```bash
docker ps | grep wikijs
docker logs wikijs
```

## ✅ Part 4: Testing and Configuration

### 8. Check container connectivity
```bash
# Verify containers can communicate
docker exec wikijs-db psql -U wikijs -d wiki -c "\dt"

# Check WikiJS application logs
docker logs wikijs --tail 20
```

### 9. Access WikiJS web interface
Open your web browser and navigate to:
```
http://localhost:3000
```

### 10. Complete WikiJS installation via web browser
1. **Administrator Account Setup**
   - Enter administrator email address
   - Set administrator password
   - Click "Install"

2. **General Settings**
   - Set site title (e.g., "My Wiki")
   - Set site description
   - Choose site URL format

3. **Storage Configuration**
   - Select "Local File System" for content storage
   - Keep default path: `/wiki/data`

4. **Authentication**
   - Configure local authentication or connect external providers
   - Complete the setup wizard

### 11. Verify installation success
- Navigate through the wiki interface
- Create a test page
- Verify data persistence by restarting containers:
```bash
docker restart wikijs wikijs-db
# Wait 30 seconds, then refresh browser
```

## ✅ Part 5: Container Management

### 12. Stop WikiJS services
```bash
docker stop wikijs wikijs-db
```

### 13. Start WikiJS services
```bash
docker start wikijs-db
sleep 10
docker start wikijs
```

### 14. Remove WikiJS containers (optional)
```bash
# Stop and remove containers (data will persist)
docker stop wikijs wikijs-db
docker rm wikijs wikijs-db

# Remove network
docker network rm wikijs-network
```

## 📋 Quick Reference

### Container Information
- **WikiJS Container**: `wikijs`
- **Database Container**: `wikijs-db`
- **Network**: `wikijs-network`
- **Web Access**: `http://localhost:3000`

### Persistent Data Locations
- **PostgreSQL Data**: `~/wikijs/data/postgres/`
- **WikiJS Config**: `~/wikijs/config/`
- **Wiki Content**: Stored in PostgreSQL database

### Useful Commands
```bash
# View logs
docker logs wikijs
docker logs wikijs-db

# Execute commands in containers
docker exec -it wikijs sh
docker exec -it wikijs-db psql -U wikijs -d wiki

# Backup database
docker exec wikijs-db pg_dump -U wikijs wiki > wiki-backup.sql
```

## 🔧 Troubleshooting

- **WikiJS won't start**: Check database container is running first
- **Connection refused**: Ensure port 3000 is not in use by another service
- **Database connection failed**: Verify environment variables match between containers
- **Data not persisting**: Check volume mount paths exist and have proper permissions

---

*Guide tested with WikiJS 2.x and PostgreSQL 15*