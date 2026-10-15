# Food Delivery API

A REST API for a food-delivery service, written in Go with the [Gin](https://github.com/gin-gonic/gin) framework. It covers user registration and login with JWT tokens, restaurant search and menus, favourites, a shopping cart, and order placement with delivery-person assignment.

Data is held in memory and seeded at start-up, so there is nothing to set up besides Go.

## Features

- **Authentication.** Register and log in; login returns a JWT (HS256) valid for one hour. Cart and order endpoints require it.
- **API key.** Every endpoint except `GET /listofRestaurants` requires an `API-Key` header.
- **Browse.** List restaurants, search by dish name (case-insensitive), view a menu, filter by food type (veg, non-veg, vegan), and list restaurants that have a favourite dish.
- **Favourites.** Toggle a dish's favourite flag.
- **Cart and orders.** Add items to a cart, view it, place an order and check its status. Placing an order assigns the first available delivery person.

## Run it

Requires Go 1.21 or later.

```bash
go mod tidy      # fetches dependencies
go run .         # serves on http://localhost:8080
```

## Endpoints

All requests need the headers `Content-Type: application/json` and `API-Key: key`, except `GET /listofRestaurants`, which needs none. Endpoints marked **JWT** also need `Authorization: <token>` (the raw token from `/login`, without a `Bearer` prefix).

| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| GET | `/listofRestaurants` | none | All restaurants with their menus |
| POST | `/register` | API key | Create a user (`{"username", "password"}`); 409 if the name is taken |
| POST | `/login` | API key | Returns a JWT |
| GET | `/search?fooditem=<text>` | API key | Restaurants with a matching dish |
| GET | `/menu/:name` | API key | Menu of one restaurant |
| GET | `/foodtype/:foodtype` | API key | Dishes of a given food type |
| GET | `/favsearch` | API key | Restaurants with at least one favourite dish |
| PATCH | `/update/:name/:dish` | API key | Toggle the favourite flag on a dish |
| POST | `/add-to-cart/:user_id` | JWT | Add an item to the user's cart |
| GET | `/view-cart/:user_id` | JWT | View the cart and its total |
| POST | `/order` | JWT | Place an order; returns an order ID |
| GET | `/order/:order_id` | JWT | Order total, status and delivery person |

## Example session

```bash
# 1. Log in with a seeded user and keep the token
curl -s -X POST localhost:8080/login \
  -H "Content-Type: application/json" -H "API-Key: key" \
  -d '{"username":"user1","password":"password1"}'

TOKEN=<paste the token from the response>

# 2. Search for a dish
curl -s "localhost:8080/search?fooditem=pizza" -H "API-Key: key"

# 3. Add to the cart
curl -s -X POST localhost:8080/add-to-cart/user1 \
  -H "Content-Type: application/json" -H "API-Key: key" -H "Authorization: $TOKEN" \
  -d '{"dish":"pizza","quantity":2,"price":10.99}'

# 4. Place an order
curl -s -X POST localhost:8080/order \
  -H "Content-Type: application/json" -H "API-Key: key" -H "Authorization: $TOKEN" \
  -d '{"restaurant_name":"abc","items":[{"dish":"burger","quantity":2,"price":7.99}]}'

# 5. Check its status (use the order_id returned above)
curl -s localhost:8080/order/ORDER-1 -H "API-Key: key" -H "Authorization: $TOKEN"
```

## Seed data

- **Users:** `user1` / `password1`, `user2` / `password2`
- **Restaurants:** `abc` (open, delivers 10:00–16:00) and `xyz` (marked closed), each with a breakfast, lunch and dinner dish
- **Delivery people:** two, both available at start

## Code layout

Everything is in `main.go`: the data types and seed data, the handlers, the API-key and JWT middleware, and the routes in `main()`.

## Scope and limitations

This is a prototype, built to practise REST design, middleware and token authentication in Go. These limits apply when reading it as more than that:

- **In-memory storage.** All data resets when the server restarts, and there is no database.
- **Demo-grade security.** Passwords are stored and compared as plain text, and the JWT secret and API key are hard-coded. A real service would hash passwords (for example with bcrypt) and read secrets from the environment.
- **Carts are not tied to the logged-in user.** Any valid token can read or change any `user_id`'s cart, and placing an order does not use or clear the cart.
- **The client supplies prices.** Order totals are computed from the prices in the request, not from a menu price list.
- **Delivery people are never released.** Each order marks one as unavailable permanently, so after two orders new orders fail with "No available delivery person".
- **Order status is derived, not stored.** `GET /order/:order_id` reports "Out for Delivery" whenever a delivery person is assigned.
- **Restaurant status and delivery hours are not enforced** when an order is placed.
- **No locking.** Handlers share global slices and maps without a mutex, so concurrent writes are not safe.
- **No automated tests.**

## Built with

Go, [Gin](https://github.com/gin-gonic/gin), and [dgrijalva/jwt-go](https://github.com/dgrijalva/jwt-go) for tokens. Endpoints were checked manually with Postman.
