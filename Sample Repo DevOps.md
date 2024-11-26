## 1. Containerize Our MERN Application

`My repo for simple MERN app` [Simple-mern-app](https://github.com/umer2k200/simple-mern-app)


### Dockerize the MERN app:

**Steps Taken:**

- Clone the repo and make it ready:
```bash
git clone https://github.com/umer2k200/simple-mern-app.git
cd simple-mern-app
cd client 
npm install
cd ../server
npm install
touch .env
```
- Add the port and mongo_url in the .env
```bash
PORT=5000
MONGO_URL=(your url)
```

- Now, create the Dockerfile and .dockerignore files for both client and server.

- server/Dockerfile:
```dockerfile

FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

- client/Dockerfile:

```dockerfile

FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

- docker-compose.yaml: (In the root directory)

```yaml
services:
    client:
        build: 
            context: ./client
        ports: #maps 3000 port of the host to 80 port of the container/client Dockerfile
            - "3000:80"
        networks: #connects to the mern-net network
            - mern-net
    server:
        build:
            context: ./server
        ports:
            - "5000:5000"
        networks:
            - mern-net
        environment: #environment variables
            - MONGO_URI=mongodb://mongo:27017/mern-db
    mongo:
        image: mongo:4.4.4 #pulls the mongo image version from the docker hub
        container_name: mern-mongo #name of the container in the docker
        ports:
            - "27017:27017"
        networks:
            - mern-net
        # volumes: #mounts the data to the host
networks:
    mern-net:
```

- Ensure that the docker is running:
```bash
docker login
# check the running containers                                
docker ps -a
```

- Build the Docker images for both:
```bash
docker-compose up --build
```
- You can also push that image to Docker Hub (Optional):
- First, create a repo in Docker hub
```bash
docker tag simple-mern-app-server-image umarasdas/mernapp-client:v1
docker tag simple-mern-app-client-image umarasdas/mernapp-server:v1
docker push umarasdas/mernapp-client:v1
docker push umarasdas/mernapp-server:v1
```

- To stop the containers:
```bash
docker-compose down
```

The application would be running on port:3000 as specified in the docker-compose.yaml file. 

---
### **2. Setting Up a CI/CD Pipeline**

**Steps Taken:**

Now in the root of your folder.
Create a .github/workflows/main.yaml

main.yml:
```yaml

name: MERN CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    name: Building Docker images
    runs-on: ubuntu-latest

    steps:
      # 1. Checkout code from the repository
      - name: Checkout code
        uses: actions/checkout@v3

      # 2. Set up Docker Buildx
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      # 3. Cache Docker layers
      - name: Cache Docker layers
        uses: actions/cache@v3
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      # 4. Build Client Docker image
      - name: Build Client Docker image
        run: |
          docker build -f client/Dockerfile -t simple-mernapp-client-image:latest ./client

      # 5. Build Server Docker image
      - name: Build Server Docker image
        run: |
          docker build -f server/Dockerfile -t simple-mernapp-server-image:latest ./server

      # 6. Push Docker images
      - name: Push the Docker images
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker tag simple-mernapp-client-image:latest umarasdas/simple-mernapp-client-image:latest
          docker tag simple-mernapp-server-image:latest umarasdas/simple-mernapp-server:latest
          docker push umarasdas/simple-mernapp-client-image:latest
          docker push umarasdas/simple-mernapp-server:latest

      # 7. Create Backend .env File
      - name: Create Backend .env File
        run: |
          echo "DATABASE_URL=mongodb://mongo:27017/mern_db" >> server/.env
          echo "JWT_SECRET=your-secret-key" >> server/.env

      # 8. Install Docker Compose
      - name: Install Docker Compose
        run: |
          sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
          sudo chmod +x /usr/local/bin/docker-compose
          docker-compose version

      # 9. Deploy with Docker Compose
      - name: Deploy with Docker Compose
        run: |
          docker-compose -f docker-compose.yaml up -d
  server:
    name: Server Build and Test
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3 
        with:
          node-version: 16

      - name: Install dependencies
        run: |
          cd server
          npm install
      
      - name: Run tests
        run: |
          cd server
          npm test

  client: 
    name: Client Build and Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 16

      - name: Install dependencies
        run: |
          cd client
          npm install

      - name: Run tests
        run: |
          cd client
          npm test

      - name: Build frontend
        run: |
          cd client
          npm run build

```

Add this file in your repository as it will we hidden to other developers.

```bash
git add .
git commit -m "Update some files."
git push origin main
```

### **Outcome:**
In this guide, we implemented a CI/CD pipeline using GitHub Actions to automate testing, building, containerziation of a mern stack application and deployment. What the above pipeline will do that whenver the someone commits and pushes the code to the Github repo, the pipeline will be triggered and run. 
- The pipiline will build the Docker images for the client and server.
- Then push the images to the Docker hub.
- Then the application is deployed by using the docker-compose with the services client, server and mongoDB.
- Then it also run tests and builds for both client and server seperately.
