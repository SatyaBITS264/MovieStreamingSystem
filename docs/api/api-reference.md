# Movie Streaming System - API Reference

## 1. Overview

The application consists of two independent FastAPI microservices:

- Movie Catalog Service
- Watch History Service

Watch History communicates with Movie Catalog to validate movie IDs before saving records.

## 2. Swagger UI

The following URLs require active Kubernetes port-forward sessions:

- Movie Catalog: http://127.0.0.1:8004/docs
- Watch History: http://127.0.0.1:8003/docs

## 3. Movie Catalog Service

Version: 1.1.0

Database: `/data/movies.db`

### Data Model

- `id`: Automatically generated integer.
- `title`: String, 1–200 characters.
- `genre`: String, 1–100 characters.
- `release_year`: Integer, between 1888 and 2100.

### API Endpoints

#### GET /health

Checks service health.

Expected status: `200 OK`

Response: `{"service":"movie-catalog","status":"ok"}`

#### POST /movies

Creates a movie.

Expected status: `201 Created`

Example request:

`{"title":"Avatar","genre":"Science Fiction","release_year":2009}`

Example response:

`{"id":6,"title":"Avatar","genre":"Science Fiction","release_year":2009}`

#### GET /movies

Returns all movies ordered by ID.

Expected status: `200 OK`

#### GET /movies/{movie_id}

Returns a movie by its ID.

Expected status: `200 OK`

If the movie does not exist, returns `404 Not Found` with:

`{"detail":"Movie not found"}`

## 4. Watch History Service

Version: 1.0.0

Database: `/data/history.db`

### Data Model

- `id`: Automatically generated integer.
- `user_id`: String, 1–100 characters.
- `movie_id`: Integer greater than zero.
- `watched_at`: UTC timestamp.

### API Endpoints

#### GET /health

Checks service health.

Expected status: `200 OK`

Response: `{"service":"watch-history","status":"ok"}`

#### POST /history

Creates a watch history record.

Before saving, the service calls Movie Catalog to validate the movie ID.

Expected status: `201 Created`

Example request:

`{"user_id":"user101","movie_id":6}`

Example response:

`{"id":3,"user_id":"user101","movie_id":6,"watched_at":"2026-11-07T20:06:52.493034+00:00"}`

Possible error responses:

- `404`: Movie does not exist in the catalog.
- `502`: Unexpected response from Movie Catalog Service.
- `503`: Movie Catalog Service connection error.

#### GET /history/{user_id}

Retrieves watch history for a specific user.

Expected status: `200 OK`

Records are returned in descending order of their IDs.

Example: `GET /history/user101`

## 5. Inter-Service Communication

The Watch History Service uses this Kubernetes service URL:

`http://movie-catalog-service:8001`

Request flow:

1. Client sends a POST request to `/history`.
2. Watch History receives the user ID and movie ID.
3. Watch History calls Movie Catalog to validate the movie ID.
4. Movie Catalog returns movie details if the movie exists.
5. Watch History saves the record in its SQLite database.
6. The API returns the newly created watch history record.

## 6. API Testing Summary

The following operations were tested successfully:

- Movie Catalog: `GET /health` — 200 OK.
- Movie Catalog: `POST /movies` — 201 Created.
- Movie Catalog: `GET /movies/6` — 200 OK.
- Watch History: `GET /health` — 200 OK.
- Watch History: `POST /history` — 201 Created.
- Watch History: `GET /history/user101` — 200 OK.

Successful creation of a watch history record for movie ID 6 demonstrates communication between the two microservices.

## 7. Interactive API Documentation

FastAPI provides interactive API documentation through Swagger UI.

- Movie Catalog: `http://127.0.0.1:8004/docs`
- Watch History: `http://127.0.0.1:8003/docs`

The corresponding Kubernetes port-forward sessions must be active to access these URLs.
