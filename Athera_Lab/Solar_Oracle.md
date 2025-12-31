<h1> SOLAR_ORACLE — Building Journey</h1>

<p>
<strong>SOLAR_ORACLE</strong> is a decentralized oracle system that fetches real-world solar irradiance data
from the <strong>NREL Solar Resource API</strong> and stores it <strong>on-chain on the Aptos blockchain</strong>.
This enables trustless access to verified solar data for energy, sustainability, and DeFi use cases.
</p>

<hr/>

<h2> Project Structure</h2>

<pre>
solar-oracle/
├── server/
│   ├── src/
│   │   └── index.ts        # Oracle backend (Node.js + Aptos SDK)
│   ├── package.json
│   ├── tsconfig.json
│   └── .env
├── move/
│   ├── sources/
│   │   └── solar_oracle.move  # On-chain Move smart contract
│   └── Move.toml
└── README.md
</pre>

<hr/>

<h2> Smart Contract: <code>solar_oracle.move</code></h2>

<p>
The Move module is responsible for securely storing, validating, and exposing solar irradiance data
on-chain. Since Move does not support floating-point numbers, all values are stored as scaled integers.
</p>

<h3> Module Declaration</h3>

<pre>
module solar_oracle::solar_oracle {
    use std::signer;
    use aptos_std::table::{Self, Table};
    use aptos_framework::timestamp;
    use aptos_framework::event;
}
</pre>

<hr/>

<h3> Error Codes</h3>

<p>
Explicit error codes are used to ensure predictable failure states and safe contract execution.
</p>

<ul>
  <li><strong>E_NOT_AUTHORIZED (1)</strong> — Caller is not authorized</li>
  <li><strong>E_ALREADY_INITIALIZED (2)</strong> — Oracle already initialized</li>
  <li><strong>E_NOT_INITIALIZED (3)</strong> — Oracle not initialized</li>
  <li><strong>E_LOCATION_NOT_FOUND (4)</strong> — No data for requested location</li>
  <li><strong>E_STALE_DATA (5)</strong> — Data is outdated</li>
</ul>

<hr/>

<h3> On-Chain Structs</h3>

<ul>
  <li><strong>LocationKey</strong> — Encoded latitude & longitude</li>
  <li><strong>SolarData</strong> — DNI, GHI, Lat-Tilt, timestamp</li>
  <li><strong>SolarRegistry</strong> — Mapping of locations to data</li>
  <li><strong>DataUpdatedEvent</strong> — Emitted on update</li>
  <li><strong>OracleInitializedEvent</strong> — Emitted on initialization</li>
</ul>

<hr/>

<h3> Public & View Functions</h3>

<p>
The contract exposes multiple read-only and utility functions to consumers.
</p>

<ul>
  <li><code>update_solar_data</code></li>
  <li><code>get_solar_data</code></li>
  <li><code>has_data</code></li>
  <li><code>get_stats</code></li>
  <li><code>is_data_fresh</code></li>
  <li><code>microdegrees_to_degrees</code></li>
  <li><code>to_decimal</code></li>
  <li><code>is_suitable_for_solar</code></li>
</ul>

<hr/>

<h3> Contract Tests</h3>

<p>
All core behaviors are validated using Move unit tests.
</p>

<ul>
  <li><code>test_initialize_and_update</code></li>
  <li><code>test_multiple_locations</code></li>
  <li><code>test_suitable_for_solar</code></li>
</ul>

<hr/>

<h2> Backend Oracle — <code>index.ts</code></h2>

<p>
The backend acts as the trusted oracle executor. It fetches real-world solar data,
normalizes it, and pushes it to Aptos.
</p>

<hr/>

<h3> Configuration</h3>

<pre>
NREL_API_KEY
ORACLE_PRIVATE_KEY
MODULE_ADDRESS
NETWORK (default: devnet)
</pre>

<p>
Custom HTTP and HTTPS agents are used to enforce IPv4 networking and improve reliability.
</p>

<hr/>

<h3> Aptos Client Initialization</h3>

<p>
The oracle uses the Aptos TypeScript SDK and an Ed25519 private key to sign transactions.
</p>

<pre>
const aptos = new Aptos(new AptosConfig({ network }));
</pre>

<p>
<strong>Oracle Address:</strong> Logged on startup for verification.
</p>

<hr/>

<h3> NREL API Data Types</h3>

<p>
The NREL API provides annual average solar irradiance values.
</p>

<ul>
  <li><strong>DNI</strong> — Direct Normal Irradiance</li>
  <li><strong>GHI</strong> — Global Horizontal Irradiance</li>
  <li><strong>Latitude Tilt</strong> — Tilted panel irradiance</li>
</ul>

<hr/>

<h3> Fetching Solar Data</h3>

<p>
Data is fetched from the official NREL Solar Resource API.
</p>

<p>
<strong>Important:</strong>  
Move does not support floating-point numbers, so all values are multiplied by <code>100</code>
before storing as <code>u64</code>.
</p>

<p>
Latitude and longitude are shifted into a positive range for compatibility:
</p>

<ul>
  <li><strong>Latitude:</strong> lat + 90</li>
  <li><strong>Longitude:</strong> lon + 180</li>
</ul>

<hr/>

<h3> Pushing Data On-Chain</h3>

<p>
Solar data is sent to the Aptos blockchain via the
<code>update_solar_data</code> entry function.
</p>

<p>
Each update emits an on-chain event and records gas usage.
</p>

<hr/>

<h3> Reading Data from Chain</h3>

<p>
On-chain data can be read directly from the <code>SolarRegistry</code> resource
to verify successful updates.
</p>

<hr/>

<h3> Oracle Update Cycle</h3>

<p>
The oracle processes multiple locations sequentially with a delay to avoid API rate limits.
</p>

<pre>
San Francisco, CA
New York City, NY
Phoenix, AZ
</pre>

<hr/>

<h3> Initialization</h3>

<p>
The oracle module must be initialized once before updates can occur.
Repeated initialization attempts are safely ignored.
</p>

<hr/>

<h2> Supported Locations</h2>

<p>
<strong>Note:</strong> The NREL API currently supports <strong>US locations only</strong>.
</p>

<hr/>

<h2>🛠 Update #1 — <strong>30.12.25</strong></h2>

<h3> Successful On-Chain Updates</h3>

<ul>
  <li><strong>San Francisco:</strong> DNI 4.05, GHI 4.35, Lat Tilt 4.96</li>
  <li><strong>New York:</strong> DNI 3.79, GHI 3.93, Lat Tilt 4.62</li>
  <li><strong>Phoenix:</strong> DNI 7.35, GHI 5.78, Lat Tilt 6.68 ⭐</li>
</ul>

<hr/>

<h3> Issue Faced</h3>

<p>
The oracle initially failed due to <strong>IPv6 routing issues</strong> in Node.js,
resulting in <code>ENETUNREACH</code> and timeout errors.
</p>

<hr/>

<h3>🔧 Fixes Applied</h3>

<ul>
  <li><code>dns.setDefaultResultOrder('ipv4first')</code></li>
  <li>Forced IPv4 with <code>family: 4</code></li>
  <li><code>proxy: false</code> to avoid proxy conflicts</li>
  <li>Browser-like <strong>User-Agent</strong> header</li>
</ul>

<p>
<strong>Result:</strong>  
The oracle now reliably fetches real-world solar data and stores it
on the Aptos blockchain without failures.
</p>

<hr/>

<h2>🎯 Final Outcome</h2>

<p>
<strong>SOLAR_ORACLE</strong> successfully bridges off-chain renewable energy data
to on-chain smart contracts, enabling transparent, verifiable,
and programmable access to solar intelligence
</p>
