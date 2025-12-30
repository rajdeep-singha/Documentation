<h1>SOLAR_ORACLE -> Building Journey </h1>

Project Structure :
solar-oracle/
├── server/
│ ├── src/
│ │ └── index.ts # Oracle backend
│ ├── package.json
│ ├── tsconfig.json
│ └── .env
├── move/
│ ├── sources/
│ │ └── solar_oracle.move
│ └── Move.toml
└── README.md

<h2>Solar_pracle.move</h2>
I am writing smart contracts which can store data onchain from this api :https://developer.nrel.gov/docs/solar/solar-resource-v1/

. So basically I have this module in my contract

module solar_oracle::solar_oracle {
use std::signer;
use aptos_std::table::{Self, Table};
use aptos_framework::timestamp;
use aptos_framework::event;
}


Have few error codes , like this

 // Error codes
    const E_NOT_AUTHORIZED: u64 = 1;
    const E_ALREADY_INITIALIZED: u64 = 2;
    const E_NOT_INITIALIZED: u64 = 3;
    const E_LOCATION_NOT_FOUND: u64 = 4;
    const E_STALE_DATA: u64 = 5;


So in my whole contract , the structs I have are :
LocationKey
SolarData
SolarRegistry

DataUpdatedEvent

OracleInitializedEvent

I have few public entry fun to update ,read , check data , convert data

#[view]
update_solar_data
get_solar_data
has_data
get_stats
is_data_fresh
microdegrees_to_degrees
to_decimal
is_suitable_for_solar

#[test_only]   
#[test(oracle = @solar_oracle)]
test_initialize_and_update
test_multiple_locations
test_suitable_for_solar 

<h2>Now working on backend</h2>

<strong>index.ts</strong>

<p>configurations</p>

const NREL_API_KEY = process.env.NREL_API_KEY!;
const ORACLE_PRIVATE_KEY = process.env.ORACLE_PRIVATE_KEY!;
const MODULE_ADDRESS = process.env.MODULE_ADDRESS!;
const NETWORK = process.env.NETWORK || "devnet";
Creating HTTP agents with IPv4 forced

const httpAgent = new http.Agent({ 
  keepAlive: true, 
  family: 4,
  timeout: 60000,
});

const httpsAgent = new https.Agent({ 
  keepAlive: true, 
  family: 4,
  timeout: 60000,
  rejectUnauthorized: true,
});

<p>Innitializing Aptos Client</p>

const config = new AptosConfig({ 
  network: NETWORK as Network 
});
const aptos = new Aptos(config);
// oracle account (from private key)

const privateKey = new Ed25519PrivateKey(ORACLE_PRIVATE_KEY);
const oracleAccount = Account.fromPrivateKey({ privateKey });

console.log(`🔑 Oracle Address: ${oracleAccount.accountAddress.toString()}`);

<p> NREL API TYPES </p>

interface NRELResponse {
  outputs: {
    avg_dni: { annual: number };
    avg_ghi: { annual: number };
    avg_lat_tilt: { annual: number };
  };
}

interface SolarData {
  latitude: number;
  longitude: number;
  dni: number;  // Direct Normal Irradiance (kWh/m²/day * 100)
  ghi: number;  // Global Horizontal Irradiance (kWh/m²/day * 100)
  lat_tilt: number; // Latitude Tilt (kWh/m²/day * 100)
  timestamp: number;
}
<p>Fetch solar resource data from NREL API</p>

async function fetchSolarData(lat: number, lon: number): Promise<SolarData> {
  console.log(`📡 Fetching NREL data for lat: ${lat}, lon: ${lon}`);
  console.log(`🔑 Using API key: ${NREL_API_KEY ? NREL_API_KEY.substring(0, 8) + '...' : 'NOT SET'}`);
  
  try {
    const response = await axios.get<NRELResponse>(
      "https://developer.nrel.gov/api/solar/solar_resource/v1.json",
      {
        params: {
          api_key: NREL_API_KEY,
          lat: lat,
          lon: lon,
        },
        timeout: 60000,  // 60 second timeout
        httpAgent: httpAgent,
        httpsAgent: httpsAgent,
        proxy: false,    // Disable proxy
        headers: {
          'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
          'Accept': 'application/json',
        },
      }
    );

    const outputs = response.data.outputs;
Convert to integers (multiply by 100 to preserve 2 decimal places)

Become a member
// Move doesn’t support floats, so we store as u64

// Coordinates are shifted to positive range for u64 compatibility

const solarData: SolarData = {
      latitude: encodeLatitude(lat),    // Shifted: lat + 90
      longitude: encodeLongitude(lon),  // Shifted: lon + 180
      dni: Math.floor(outputs.avg_dni.annual * 100),
      ghi: Math.floor(outputs.avg_ghi.annual * 100),
      lat_tilt: Math.floor(outputs.avg_lat_tilt.annual * 100),
      timestamp: Math.floor(Date.now() / 1000),
    };

    console.log(`✅ NREL Data:`, {
      dni: `${outputs.avg_dni.annual} kWh/m²/day`,
      ghi: `${outputs.avg_ghi.annual} kWh/m²/day`,
      lat_tilt: `${outputs.avg_lat_tilt.annual} kWh/m²/day`,
    });

    return solarData;
  } catch (error) {
    if (axios.isAxiosError(error)) {
      console.error(`❌ NREL API Error:`, {
        code: error.code,
        message: error.message,
        status: error.response?.status,
        data: error.response?.data,
      });
      throw new Error(`NREL API Error (${error.code}): ${error.message}`);
    }
    throw error;
  }
}
<p>Pushing solar data into onchain APTOS</p>

async function pushToChain(data: SolarData): Promise<string> {
  try {
    console.log(`📤 Pushing to Aptos blockchain...`);

    const transaction = await aptos.transaction.build.simple({
      sender: oracleAccount.accountAddress,
      data: {
        function: `${MODULE_ADDRESS}::solar_oracle::update_solar_data`,
        functionArguments: [
          data.latitude,
          data.longitude,
          data.dni,
          data.ghi,
          data.lat_tilt,
          data.timestamp,
        ],
      },
    });

    const committedTxn = await aptos.signAndSubmitTransaction({
      signer: oracleAccount,
      transaction,
    });

    const executedTransaction = await aptos.waitForTransaction({
      transactionHash: committedTxn.hash,
    });

    console.log(`✅ Transaction committed: ${committedTxn.hash}`);
    console.log(`   Gas used: ${executedTransaction.gas_used}`);

    return committedTxn.hash;
  } catch (error) {
    console.error(`❌ Transaction failed:`, error);
    throw error;
  }
}
<p>Read solar data from blockchain</p>

async function readFromChain(lat: number, lon: number): Promise<void> {
  try {
    const latEncoded = encodeLatitude(lat);
    const lonEncoded = encodeLongitude(lon);

    console.log(`📖 Reading on-chain data for (${lat}, ${lon})`);
    console.log(`   Encoded: lat=${latEncoded}, lon=${lonEncoded}`);

    const resource = await aptos.getAccountResource({
      accountAddress: MODULE_ADDRESS,
      resourceType: `${MODULE_ADDRESS}::solar_oracle::SolarRegistry`,
    });

    console.log(`📖 Registry data:`, resource);
  } catch (error) {
    console.error(`❌ Failed to read on-chain data:`, error);
  }
}
<p>Main oracle update cycle</p>

async function updateOracle(locations: Array<{ lat: number; lon: number }>) {
  console.log(`\n🚀 Starting Oracle Update Cycle`);
  console.log(`   Network: ${NETWORK}`);
  console.log(`   Module: ${MODULE_ADDRESS}`);
  console.log(`   Locations: ${locations.length}`);

  for (const loc of locations) {
    try {
      console.log(`\n📍 Processing location: ${loc.lat}, ${loc.lon}`);
      
      // Fetch from NREL
      const solarData = await fetchSolarData(loc.lat, loc.lon);
      
      // Push to Aptos
      const txHash = await pushToChain(solarData);
      
      console.log(`✅ Successfully updated location ${loc.lat}, ${loc.lon}`);
      
      // Wait 2 seconds between updates to avoid rate limiting
      await new Promise(resolve => setTimeout(resolve, 2000));
      
    } catch (error) {
      console.error(`❌ Failed to update location ${loc.lat}, ${loc.lon}:`, error);
    }
  }

  console.log(`\n✅ Oracle update cycle completed\n`);
}
<p>Initialize oracle (deploy module if needed)</p>

async function initialize() {
  console.log(`\n🔧 Initializing Oracle Module...`);
  
  try {
    const transaction = await aptos.transaction.build.simple({
      sender: oracleAccount.accountAddress,
      data: {
        function: `${MODULE_ADDRESS}::solar_oracle::initialize`,
        functionArguments: [],
      },
    });

    const committedTxn = await aptos.signAndSubmitTransaction({
      signer: oracleAccount,
      transaction,
    });

    await aptos.waitForTransaction({ transactionHash: committedTxn.hash });
    
    console.log(`✅ Oracle initialized: ${committedTxn.hash}`);
  } catch (error: any) {
    if (error.message?.includes("RESOURCE_ALREADY_EXISTS")) {
      console.log(`ℹ️  Oracle already initialized`);
    } else {
      throw error;
    }
  }
}
<p>US Locations only — NREL API only supports US locations!</p>

const LOCATIONS = [
  { lat: 37.7749, lon: -122.4194 }, // San Francisco, CA
  { lat: 40.7128, lon: -74.0060 },  // New York City, NY
  { lat: 33.4484, lon: -112.0740 }, // Phoenix, AZ
];

// Run oracle
(async () => {
  try {
    // Initialize module (run once)
    // await initialize();
    
    // Update solar data
    await updateOracle(LOCATIONS);
    
    // Read back data (verification)
    await readFromChain(LOCATIONS[0].lat, LOCATIONS[0].lon);
    
  } catch (error) {
    console.error("Fatal error:", error);
    process.exit(1);
  }
})();

<h2>Update #1 (30.12.25)</h2>

<p>Issue I was facing an issue</p>

It’s working now! All three US locations were successfully updated:

San Francisco: DNI 4.05, GHI 4.35, Lat Tilt 4.96 kWh/m²/day

New York: DNI 3.79, GHI 3.93, Lat Tilt 4.62 kWh/m²/day

Phoenix: DNI 7.35, GHI 5.78, Lat Tilt 6.68 kWh/m²/day (highest!)

<p>What Fixed It</p>
The issue was IPv6 vs IPv4 routing. Node.js was trying IPv6 first (which failed with ENETUNREACH) before falling back to IPv4 (which timed out). The browser handles this differently.

<p>Key fixes applied:</p>

dns.setDefaultResultOrder(‘ipv4first’) — Force IPv4 DNS resolution first
Custom agents with family: 4 — Force IPv4 connections
proxy: false — Disable any proxy interference
Browser-like User-Agent — Some APIs filter by User-Agent
The data is now being pushed to Aptos blockchain successfully with real NREL solar irradiance data!