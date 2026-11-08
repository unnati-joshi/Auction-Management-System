# Auction Management System

A full-stack auction web application where users can create listings, place bids, and follow auction activity in real time. The project uses a React client, an Express API, MongoDB, and Socket.IO.

## Features

- Create auctions with item details, starting price, start/end dates, category, and image.
- Browse active auctions with paginated listings and open auction detail pages.
- Place bids through the API or the Socket.IO handler.
- Receive live bid updates and auction-room presence events through Socket.IO.
- Authenticate with JWTs stored in HTTP-only cookies; passwords are hashed with bcrypt.
- View user dashboards, personal auctions and bids, profiles, and admin pages.
- Upload auction images through Cloudinary signed upload flows.
- Send contact form messages using Resend when configured.
- Use React Query for server-state fetching, caching, and query refreshes; Redux Toolkit stores authentication state.

## Tech stack

| Area | Technology |
| --- | --- |
| Client | React 19, Vite 6, Tailwind CSS 4 |
| Client state | Redux Toolkit, TanStack Query |
| API | Node.js, Express 5 |
| Database | MongoDB with Mongoose |
| Real-time | Socket.IO |
| Authentication | JWT, HTTP-only cookies, bcrypt |
| Image storage | Cloudinary |
| Email | Resend |

## How bidding works

1. The client sends a bid through the authenticated REST endpoint or Socket.IO event.
2. The backend loads the auction and validates the amount, auction end time, and bidder. The seller cannot bid on their own auction.
3. The current implementation allows a bid from **₹1 to ₹10 above** the greater of the current price and starting price.
4. The backend uses a conditional `findOneAndUpdate`: it updates the auction only if its current price still matches the price read during validation and the auction is still open. If another bid changes the price first, the conditional update matches no document and the bid is rejected so the user can retry.
5. After a successful update, the server broadcasts the new auction state to users in that auction's Socket.IO room.

This is optimistic concurrency control using a conditional atomic update; the current bid path does not use a MongoDB transaction. Bid history is embedded in the auction document, which keeps an auction and its bids together but may need reconsideration for very high-volume auctions.

## Authentication and uploads

Signup hashes passwords with bcrypt. Successful login issues a JWT in an HTTP-only cookie. Protected API routes verify the token, and Socket.IO authenticates connections from the same cookie before registering auction handlers.

Auction image uploads use Cloudinary. The backend creates signed upload parameters and stores an upload session, which the auction creation flow checks before saving the listing.

## Local development

### Requirements

- Node.js and npm
- A MongoDB connection string
- Cloudinary credentials for image uploads
- A Resend API key if contact email delivery is needed

### Configure the server

```bash
cd server
npm install
```

Create `server/.env` with the environment variables below, then run:

```bash
npm run dev
```

The default server port is `4000`.

### Configure the client

In another terminal:

```bash
cd client
npm install
npm run dev
```

Vite prints the local client address when it starts.

## Environment variables

The server reads these variables from `server/.env`:

| Variable | Required | Purpose |
| --- | --- | --- |
| `MONGO_URL` | Yes | MongoDB connection string |
| `JWT_SECRET` | Yes | Secret used to sign and verify JWTs |
| `ORIGIN` | Yes | Allowed client origin for CORS and Socket.IO |
| `PORT` | No | Server port; defaults to `4000` |
| `JWT_EXPIRES_IN` | No | JWT lifetime; defaults to `7d` |
| `COOKIE_DOMAIN` | No | Cookie domain for cross-subdomain deployments |
| `CLOUDINARY_CLOUD_NAME` | For image upload | Cloudinary account name |
| `CLOUDINARY_API_KEY` | For image upload | Cloudinary API key |
| `CLOUDINARY_API_SECRET` | For image upload | Cloudinary API secret |
| `CLOUDINARY_URL` | Optional | Cloudinary connection URL, if used by the deployment |
| `RESEND_API_KEY` | For email | Resend API key |

Keep `.env` files and production secrets out of version control.

## Project layout

```text
Auction-Management-System/
├── client/
│   └── src/
│       ├── components/       # Shared and landing-page components
│       ├── hooks/            # React Query and Socket.IO hooks
│       ├── pages/            # Auctions, account, dashboard, and admin pages
│       ├── services/         # API service functions
│       ├── store/            # Redux Toolkit state
│       └── config/           # API and Socket.IO clients
└── server/
    ├── config/               # Environment and database setup
    ├── controllers/          # Request handling and application logic
    ├── middleware/           # Authentication and authorization
    ├── models/               # Mongoose schemas
    ├── routes/               # REST route definitions
    ├── services/             # Cloudinary integrations
    ├── socket/               # Socket.IO setup and auction events
    └── utils/                # JWT, cookies, and supporting helpers
```

## Deployment notes

The repository contains Vercel configuration for the client and server, as well as a long-running server entry point. The Socket.IO bidding experience requires persistent connections. For a multi-instance deployment, configure a shared Socket.IO adapter such as Redis so events can reach users connected to different instances. Confirm the chosen hosting platform supports the persistent connections required by Socket.IO.

## Known limitations and next steps

- Automated tests are not currently configured; the server's `test` script is a placeholder.
- Winner assignment is currently set when an expired auction is requested and has bids but no saved winner. A scheduled worker would finalize expired auctions independently of page visits.
- Bid history is stored inside each auction document. Separating bids into their own collection may help with very large histories and reporting.
- Socket.IO room presence is held in process memory; multi-instance deployments need shared coordination.
- Add rate limiting and stronger bidding abuse controls, plus monitoring and structured logging.
- Add pagination or cursor-based pagination to other growing collections where needed.
- Complete auction edit/cancel flows and define what happens to active bids when an auction changes.

## License

MIT. See `LICENSE`.
