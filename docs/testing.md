# Testing

This document explains how the backend is tested. It covers the strategy, the folder layout, the hotel search suites and the commands.

**Scope:** the suite descriptions in this document cover only the hotel search module and the Ascenda client, the tests listed under "My contribution" in the README. Other team members wrote the remaining suites (authentication, payments, profile, hotel details and rooms). The folder layout and the commands apply to all of them, but this document does not describe their contents.

- [1. Strategy](#1-strategy)
- [2. Folder layout](#2-folder-layout)
- [3. Running the tests](#3-running-the-tests)
- [4. Hotel search suites](#4-hotel-search-suites)
- [5. Other testing by the team](#5-other-testing-by-the-team)
- [6. Gaps](#6-gaps)

## 1. Strategy

The project uses Jest. There are two kinds of automated test.

| Kind | Method | What it checks |
|---|---|---|
| Unit test | White-box, with path coverage. | One file at a time. The test replaces its neighbours with mocks. |
| Integration test | Black-box, bottom-up, with robust equivalence classes. | Several layers together. Only the outside world (HTTP calls, the database) is replaced. |

A *robust equivalence class* test picks one input from each valid class and one from each invalid class of every field. A request body has many fields. Each field has valid and invalid classes, for example a missing value, a wrong type or an out-of-range value.

Both kinds of test run without the real Ascenda API or Stripe. Sample Ascenda responses are in `test/mocks/`.

## 2. Folder layout

```
test/
  unit_tests/          20 files, mostly one for each source file
  integration_tests/   16 files on main (10 on the caching branch)
  mocks/               hotel3132ApiMocks.js, userData.js
  setup/
    jest.setup.js      Test environment variables and timeout
    testDb.js          In-memory MongoDB helper
```

`jest.setup.js` sets safe defaults for `JWT_SECRET` and `STRIPE_SECRET_KEY`. It sets a 30 second timeout and silences `console.log`. `testDb.js` starts a `mongodb-memory-server` instance for tests that need a database.

The counts on `main` are about 240 unit test cases and about 180 integration test cases. The counts come from the `test(...)` and `it(...)` blocks in the files.

## 3. Running the tests

1. Run `npm install`.
2. Run `npm test`.

To run one suite, pass a path:

```
npm test -- test/unit_tests/hotelResultsService.unit.test.js
```

The first run of a database test downloads a MongoDB binary for `mongodb-memory-server`.

## 4. Hotel search suites

These suites cover the hotel search module and the Ascenda client. They hold about 160 test cases.

| Suite | Cases | Layer under test | How the test isolates it |
|---|---|---|---|
| `hotelResultsRouter.unit.test.js` | 3 | Router | Mocks the controller. |
| `hotelResultsController.unit.test.js` | 17 | Controller | Mocks the service. |
| `hotelResultsService.unit.test.js` | 36 | Service | Mocks `ascendaApiCaller`. Uses the real `safeAssign`. |
| `utils2.test.js` | 19 | Ascenda client | Mocks the poller and `fetch`. |
| `hotelResultsRouter.integration.test.js` | 12 | Router and controller and service | Supertest sends real HTTP requests. `fetch` is mocked. |
| `hotelResultsController.integration.test.js` | 34 | Controller and service and client | `fetch` is mocked. |
| `hotelResultsService.integration.test.js` | 28 | Service and client | `fetch` is mocked. |
| `utils2.integration.test.js` | 11 | Ascenda client and poller | `fetch` is mocked. |

### What the unit tests check

**Controller**

- Returns `400` for an empty body, a missing field, a wrong type, a bad filter object, a bad sort option, a bad date, `checkout` on or before `checkin`, and `checkin` less than 3 days ahead.
- Returns `200` with the data, or with an empty list.
- Builds the DTOs and the guest string. Sets `sort` and `filters` to `null` when they are not requested.
- Returns `500` when the service throws.

**Router**

- A `POST` reaches the controller with `req`, `res` and `next`.
- Other methods are not handled.

**Service**

- The result has a `data` field. Fully matched hotels have static fields.
- A hotel with no price is dropped. A hotel with no static data keeps only `id`, `price`, `rating` and `score`.
- Defaults apply: rating `2.5`, score `70`, image URL `null` or prefix plus `1.jpg`.
- Price, rating and score filters work alone and together.
- Sorting works for price, rating and score, in both directions. An unknown sort variable leaves the order unchanged.
- Filters and sorting work together for every combination.
- The service throws when the caller throws, or when `data1` or `data2` has the wrong shape. It returns an empty list for an empty price list.

**Ascenda client**

- The URL builder produces the right URL for `1` and `2`. It throws for any other value.
- The caller uses the poller for `1` and `fetch` for `2`.
- Errors surface for: poller max attempts, HTTP `500`, non-JSON bodies.
- `safeAssign` returns `null` for `undefined`, `null` and `""`. It keeps `0`, `false`, strings, objects and arrays.

### What the integration tests check

- **Router.** `GET` returns `404`. Bad bodies return `400`. A fetch failure and a poller timeout return `500`. An empty Ascenda result returns `200` with an empty list. Missing static data returns `200` with default fields.
- **Controller.** Request validation, date rules (with a fixed clock), filter-only, sort-only and filter-plus-sort requests for every key and direction. Third-party failures: failed fetch, poller max attempts, invalid JSON, a price list that is null or not an array, a static list that is null or not an array. Edge cases: an empty static list and an empty price list.
- **Service.** The same filter and sort combinations, run against mocked HTTP responses instead of a mocked caller. Null prices, hotels missing from either list, and defaults.
- **Ascenda client.** Success, retry after an incomplete poll, HTTP `500`, invalid JSON, a network error and the maximum number of poll attempts.

## 5. Other testing by the team

The project report describes two more kinds of testing. Their code is not in this repository.

| Kind | Description |
|---|---|
| Fuzzing | The team sent every string of the Big List of Naughty Strings (516 inputs) to each field of the `/hotelresults` and `/combined-hotel-data` bodies. They used the Postman collection runner. The endpoints returned `400` with an error message, as intended. |
| End-to-end | Cypress tests in the front end cover the flow from login to booking, and the cycle of sign-up, login and account deletion. |

## 6. Gaps

- No test covers the search cache. The hotel search suites were written for the service without the cache.
- The service suites mock the Ascenda client or `fetch`. They do not mock the cache model. The authors wrote them against the `main` service, so use `main` to run them.
- The tests use mocked Ascenda responses. They do not call the real API.
