# redfish-interface
## Intro
The redfish interface is a tool that can interface with one or more devices via the RedFish API for monitoring, logging, and basic management tasks. It has an API service that can receive REST calls making queries about the status of either a specific piece of hardware, or all the pieces of hardware. It can also receive calls requesting that basic tasks be performed on the boards under management, such as power on/power off/reboot.

The API server will serve the status queries by referencing a Postgres DB that is regularly populated by a worker that collects data every second. In the beginning, we will support ~20 specific fields (TBD) and the remaining data will be placed in blob storage. The PSQL row will link to the blob storage entry.

The Postgres DB will be fairly simple: there will be one table with a composite primary key consisting of the board ID + timestamp. Each row will contain the basic fields + the link to the blob storage. This will amount to approx. 1 byte per char/20 chars per entry/20 entries --> 400 bytes, or ~0.5kB per entry. 86,000 seconds in a day + 30 days ~~~ 30 * 100000 * 0.5 kB = 300000 * 0.5 kB = 150000kB = 150mB per board. Even if our rows are 2x the size we estimated, this is 300MB per board, so for every 3-6 boards, we would fill up 1 GB of space. If this gets scaled up in the future, we could consider partitioning the table on board ID, but for now, a single partition is sufficient.

Basic tasks (power on/off, restart) will be handled directly by the API server, which will send commands to the board specified. The addresses of the boards will be stored in a REDIS cache, keyed by their ID. We might use GRPC for these requests, but the underlying calls to RedFish will have to be REST.

To populate the DB with information, we will have a worker that fetches the information every second and writes it to the DB. In the future, we will add a cleanup worker to move data past 1 month of age to cold storage. After 1 year, it will be deleted from cold storage.
