## Test Dockerfiles

### Frontend 

cd Frontend
docker build -t nivala-frontend .
docker run -p 5173:80 nivala-frontend

### Backend

cd Backend
docker build -t nivala-backend .
*docker run -p 4000:4000 --env-file .env nivala-backend

### Admin

cd Admin
docker build -t nivala-admin .
*docker run -p 5174:80 nivala-admin

* = Needs MongoDB to run