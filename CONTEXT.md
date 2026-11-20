# meli

A read-only reader for MercadoLibre Chile: search live listings, look up one listing, walk categories, list trending keywords. It exists so that both a person at a terminal and an agent needing machine output can see the same marketplace facts without the marketplace's storefront.

## Language

### Marketplace

**Site**:
The country-scoped marketplace a listing belongs to. This project serves one site only, `MLC` (MercadoLibre Chile).
_Avoid_: market, region, country, domain

**Listing**:
A single thing offered for sale on the site, with a price, condition, stock and a seller. The only marketplace object this CLI reads in bulk.
_Avoid_: item, product, article, publicación

**Listing ID**:
The site-prefixed identifier of a listing, e.g. `MLC1234567890`. It is the handle a lookup takes; it is not a listing.
_Avoid_: item id, product id, SKU

**Category**:
A node in the site's classification tree that listings are filed under. Categories nest; the site root is a category too.
_Avoid_: department, section, tag

**Trending keyword**:
A search phrase the site reports as currently popular, optionally narrowed to one category. It is not a listing and carries no ids.
_Avoid_: trend, suggestion, hot word

**Seller**:
The marketplace account that published a listing. Public reputation is visible; nothing else about a seller is in scope.
_Avoid_: vendor, merchant, store, shop

### Identity and access

**Account**:
The MercadoLibre user whose authorization this CLI holds, and therefore the identity every read is performed as. Distinct from a seller: the same account may also be one.
_Avoid_: user, owner, login

**Application**:
The registered Meli integration that identifies this CLI to the marketplace and defines which permissions it may request. One application, created once by a person, read-scoped by design.
_Avoid_: app, client, integration, credentials

**Grant**:
The authorization a person gives an application to act for their account, and the token pair that results from it. A grant is per account and per application.
_Avoid_: login, session, connection

**Paste-code login**:
The grant flow where the person authorizes in a browser on any machine and hands the resulting code back to the CLI. Chosen so a headless box needs no browser and no end-user secret handling.
_Avoid_: device flow, callback flow, OAuth dance

**Access token**:
The short-lived credential presented on every read. Expires in hours.
_Avoid_: session token, API key

**Refresh token**:
The long-lived, single-use credential that mints the next access token. Once exchanged it is dead, which makes losing or reusing one a fatal, non-retryable condition.
_Avoid_: renew token, long-lived token

**Token store**:
The one place a grant's credentials live on a machine — the OS keyring when available, a permission-restricted file when not. Never a tracked path, a shell profile, or a log.
_Avoid_: keychain, vault, secrets file, config

### Shape of the tool

**Read-only posture**:
The property that the CLI cannot mutate a marketplace account at all, by construction rather than by convention: no write command, no write-capable request path, and a read-scoped application on the marketplace side.
_Avoid_: no-write mode, safety flag, read-only setting

**Data plane**:
The set of marketplace reads the CLI performs as the account. Structurally limited to retrievals.
_Avoid_: API, backend, upstream

**Token plane**:
The credential-exchange calls that keep a grant alive, kept apart from the data plane so that the only outbound non-retrieval in the system is the exchange itself.
_Avoid_: auth service, OAuth layer

**Paging**:
The window of a result set a single search returns: how many, from where, and how many exist in total. Every result count the CLI reports is the site's own, never an estimate.
_Avoid_: pagination, cursor, page size

**CLP**:
Chilean pesos, the currency the site prices in, with no minor unit — amounts are whole numbers and are never rounded by this CLI.
_Avoid_: pesos, currency unit
