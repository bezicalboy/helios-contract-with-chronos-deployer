import time
import json

from web3 import Web3

from solcx import compile_source, install_solc, set_solc_version

from dotenv import load_dotenv

# --- Configuration ---

load_dotenv()

RPC_URL = "https://testnet1.helioschainlabs.org"

CHAIN_ID = 42000

CHRONOS_CONTRACT_ADDRESS = "0x0000000000000000000000000000000000000830"

# --- Chronos Contract ABI ---

CHRONOS_ABI_STRING = '''

[{"inputs":[{"internalType":"address","name":"contractAddress","type":"address"},{"internalType":"string","name":"abi","type":"string"},{"internalType":"string","name":"methodName","type":"string"},{"internalType":"string[]","name":"params","type":"string[]"},{"internalType":"uint64","name":"frequency","type":"uint64"},{"internalType":"uint64","name":"expirationBlock","type":"uint64"},{"internalType":"uint64","name":"gasLimit","type":"uint64"},{"internalType":"uint256","name":"maxGasPrice","type":"uint256"},{"internalType":"uint256","name":"amountToDeposit","type":"uint256"}],"name":"createCron","outputs":[{"internalType":"bool","name":"success","type":"bool"}],"stateMutability":"nonpayable","type":"function"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"fromAddress","type":"address"},{"indexed":true,"internalType":"address","name":"toAddress","type":"address"},{"indexed":false,"internalType":"uint64","name":"cronId","type":"uint64"}],"name":"CronCreated","type":"event"}]

'''

CHRONOS_ABI = json.loads(CHRONOS_ABI_STRING)

# --- Web3 Setup ---

w3 = Web3(Web3.HTTPProvider(RPC_URL))

if not w3.is_connected():

    raise ConnectionError(f"Failed to connect to the RPC URL: {RPC_URL}")

# --- Helper Functions ---

def get_dynamic_fee_params():

    """Gets dynamic EIP-1559 fee parameters."""import time

import json

from web3 import Web3

from solcx import compile_source, install_solc, set_solc_version

from dotenv import load_dotenv

# --- Configuration ---

load_dotenv()

RPC_URL = "https://testnet1.helioschainlabs.org"

CHAIN_ID = 42000

CHRONOS_CONTRACT_ADDRESS = "0x0000000000000000000000000000000000000830"

# --- Chronos Contract ABI ---

CHRONOS_ABI_STRING = '''

[{"inputs":[{"internalType":"address","name":"contractAddress","type":"address"},{"internalType":"string","name":"abi","type":"string"},{"internalType":"string","name":"methodName","type":"string"},{"internalType":"string[]","name":"params","type":"string[]"},{"internalType":"uint64","name":"frequency","type":"uint64"},{"internalType":"uint64","name":"expirationBlock","type":"uint64"},{"internalType":"uint64","name":"gasLimit","type":"uint64"},{"internalType":"uint256","name":"maxGasPrice","type":"uint256"},{"internalType":"uint256","name":"amountToDeposit","type":"uint256"}],"name":"createCron","outputs":[{"internalType":"bool","name":"success","type":"bool"}],"stateMutability":"nonpayable","type":"function"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"fromAddress","type":"address"},{"indexed":true,"internalType":"address","name":"toAddress","type":"address"},{"indexed":false,"internalType":"uint64","name":"cronId","type":"uint64"}],"name":"CronCreated","type":"event"}]

'''

CHRONOS_ABI = json.loads(CHRONOS_ABI_STRING)

# --- Web3 Setup ---

w3 = Web3(Web3.HTTPProvider(RPC_URL))

if not w3.is_connected():

    raise ConnectionError(f"Failed to connect to the RPC URL: {RPC_URL}")

# --- Helper Functions ---

def get_dynamic_fee_params():

    """Gets dynamic EIP-1559 fee parameters."""

    latest_block = w3.eth.get_block('latest')

    base_fee = latest_block['baseFeePerGas']

    priority_fee = w3.to_wei(2, 'gwei')

    max_fee_per_gas = base_fee + priority_fee

    return {"maxPriorityFeePerGas": priority_fee, "maxFeePerGas": max_fee_per_gas}

def sign_and_send_transaction(tx, account):

    """Signs and sends a transaction."""

    tx['nonce'] = w3.eth.get_transaction_count(account.address)

    tx['gas'] = w3.eth.estimate_gas(tx)

    signed_tx = account.sign_transaction(tx)

    tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)

    print(f"Transaction sent. Waiting for receipt... Hash: {tx_hash.hex()}")

    tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=180)

    print("✅ Transaction confirmed!")

    return tx_receipt

def deploy_and_schedule(account):

    """Deploys a simple counter contract and schedules a cron job for it."""

    # 1. --- Define and Deploy the Simple Contract ---

    print("\n--- Step 1: Deploying the 'SuccessCounter' contract ---")

    solidity_source = """

        pragma solidity ^0.8.9;

        contract SuccessCounter {

            uint256 public count;

            address public lastCaller;

            event Incremented(uint256 newCount, address indexed caller);

            function increment() public {

                count++;

                lastCaller = msg.sender;

                emit Incremented(count, msg.sender);

            }

        }

    """

    compiled_sol = compile_source(solidity_source, output_values=["abi", "bin"])

    contract_id, contract_interface = compiled_sol.popitem()

    contract_abi = contract_interface["abi"]

    contract_bytecode = contract_interface["bin"]

    Contract = w3.eth.contract(abi=contract_abi, bytecode=contract_bytecode)

    deploy_tx = Contract.constructor().build_transaction({

        "from": account.address, "chainId": CHAIN_ID, **get_dynamic_fee_params()

    })

    deploy_receipt = sign_and_send_transaction(deploy_tx, account)

    deployed_address = deploy_receipt.contractAddress

    print(f"✅ 'SuccessCounter' deployed at: {deployed_address}")

    # 2. --- Schedule the Chronos Cron Job ---

    print("\n--- Step 2: Scheduling a cron job with a 10 HLS deposit ---")

    chronos_contract = w3.eth.contract(address=w3.to_checksum_address(CHRONOS_CONTRACT_ADDRESS), abi=CHRONOS_ABI)

    

    current_block = w3.eth.block_number

    # Schedule it to run for the first time in ~2 minutes (at 10s block times)

    next_exec_block = current_block + 12

    # Let it expire far in the future

    expiration_block = current_block + 5000 

    create_cron_tx = chronos_contract.functions.createCron(

        w3.to_checksum_address(deployed_address),

        json.dumps(contract_abi),

        "increment",  # Method name to call

        [],           # No parameters for the 'increment' function

        50,           # Frequency: run every 50 blocks

        expiration_block,

        200000,       # Gas limit for the call

        w3.to_wei(10, 'gwei'),  # Max gas price

        w3.to_wei(10, 'ether')  # Deposit: 10 HLS

    ).build_transaction({

        "from": account.address, "chainId": CHAIN_ID, **get_dynamic_fee_params()

    })

    schedule_receipt = sign_and_send_transaction(create_cron_tx, account)

    

    # --- Extract cronId from the event log ---

    cron_created_event = chronos_contract.events.CronCreated().process_receipt(schedule_receipt)

    if cron_created_event:

        cron_id = cron_created_event[0]['args']['cronId']

        print(f"\n✅✅✅ Cron job successfully created! Cron ID: {cron_id}")

    else:

        print("\n⚠️ Cron job created, but could not retrieve the Cron ID from logs.")

    print("\n--- Verification ---")

    print(f"The cron will first attempt to run at or after block: {next_exec_block}")

    print("Check a block explorer for your new contract address to see the 'count' value increase after that block has passed.")

if __name__ == "__main__":

    try:

        install_solc('0.8.9')

        set_solc_version('0.8.9')

    except Exception as e:

        print(f"Solidity compiler setup error: {e}")

    try:

        with open("private_keys.txt", "r") as f:

            private_key = f.readline().strip()

        account = w3.eth.account.from_key(private_key)

        print(f"--- Loaded Account: {account.address} ---")

        deploy_and_schedule(account)

    except FileNotFoundError:

        print("❌ Error: `private_keys.txt` not found.")

    except Exception as e:

        print(f"An unexpected error occurred: {e}")

    latest_block = w3.eth.get_block('latest')

    base_fee = latest_block['baseFeePerGas']

    priority_fee = w3.to_wei(2, 'gwei')

    max_fee_per_gas = base_fee + priority_fee

    return {"maxPriorityFeePerGas": priority_fee, "maxFeePerGas": max_fee_per_gas}

def sign_and_send_transaction(tx, account):

    """Signs and sends a transaction."""

    tx['nonce'] = w3.eth.get_transaction_count(account.address)

    tx['gas'] = w3.eth.estimate_gas(tx)

    signed_tx = account.sign_transaction(tx)

    tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)

    print(f"Transaction sent. Waiting for receipt... Hash: {tx_hash.hex()}")

    tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=180)

    print("✅ Transaction confirmed!")

    return tx_receipt

def deploy_and_schedule(account):

    """Deploys a simple counter contract and schedules a cron job for it."""

    # 1. --- Define and Deploy the Simple Contract ---

    print("\n--- Step 1: Deploying the 'SuccessCounter' contract ---")

    solidity_source = """

        pragma solidity ^0.8.9;

        contract SuccessCounter {

            uint256 public count;

            address public lastCaller;

            event Incremented(uint256 newCount, address indexed caller);

            function increment() public {

                count++;

                lastCaller = msg.sender;

                emit Incremented(count, msg.sender);

            }

        }

    """

    compiled_sol = compile_source(solidity_source, output_values=["abi", "bin"])

    contract_id, contract_interface = compiled_sol.popitem()

    contract_abi = contract_interface["abi"]

    contract_bytecode = contract_interface["bin"]

    Contract = w3.eth.contract(abi=contract_abi, bytecode=contract_bytecode)

    deploy_tx = Contract.constructor().build_transaction({

        "from": account.address, "chainId": CHAIN_ID, **get_dynamic_fee_params()

    })

    deploy_receipt = sign_and_send_transaction(deploy_tx, account)

    deployed_address = deploy_receipt.contractAddress

    print(f"✅ 'SuccessCounter' deployed at: {deployed_address}")

    # 2. --- Schedule the Chronos Cron Job ---

    print("\n--- Step 2: Scheduling a cron job with a 10 HLS deposit ---")

    chronos_contract = w3.eth.contract(address=w3.to_checksum_address(CHRONOS_CONTRACT_ADDRESS), abi=CHRONOS_ABI)

    

    current_block = w3.eth.block_number

    # Schedule it to run for the first time in ~2 minutes (at 10s block times)

    next_exec_block = current_block + 12

    # Let it expire far in the future

    expiration_block = current_block + 5000 

    create_cron_tx = chronos_contract.functions.createCron(

        w3.to_checksum_address(deployed_address),

        json.dumps(contract_abi),

        "increment",  # Method name to call

        [],           # No parameters for the 'increment' function

        50,           # Frequency: run every 50 blocks

        expiration_block,

        200000,       # Gas limit for the call

        w3.to_wei(10, 'gwei'),  # Max gas price

        w3.to_wei(10, 'ether')  # Deposit: 10 HLS

    ).build_transaction({

        "from": account.address, "chainId": CHAIN_ID, **get_dynamic_fee_params()

    })

    schedule_receipt = sign_and_send_transaction(create_cron_tx, account)

    

    # --- Extract cronId from the event log ---

    cron_created_event = chronos_contract.events.CronCreated().process_receipt(schedule_receipt)

    if cron_created_event:

        cron_id = cron_created_event[0]['args']['cronId']

        print(f"\n✅✅✅ Cron job successfully created! Cron ID: {cron_id}")

    else:

        print("\n⚠️ Cron job created, but could not retrieve the Cron ID from logs.")

    print("\n--- Verification ---")

    print(f"The cron will first attempt to run at or after block: {next_exec_block}")

    print("Check a block explorer for your new contract address to see the 'count' value increase after that block has passed.")

if __name__ == "__main__":

    try:

        install_solc('0.8.9')

        set_solc_version('0.8.9')

    except Exception as e:

        print(f"Solidity compiler setup error: {e}")

    try:

        with open("private_keys.txt", "r") as f:

            private_key = f.readline().strip()

        account = w3.eth.account.from_key(private_key)

        print(f"--- Loaded Account: {account.address} ---")

        deploy_and_schedule(account)

    except FileNotFoundError:

        print("❌ Error: `private_keys.txt` not found.")

    except Exception as e:

        print(f"An unexpected error occurred: {e}")Chronos Smart Contract Scheduler

This script demonstrates how to programmatically deploy a simple smart contract and then schedule a recurring function call (a "cron job") using the Chronos protocol on the Helios testnet.

The script performs two main actions:

Deploys SuccessCounter.sol: A simple contract that has a public count variable and an increment() function.

Schedules a Cron Job: It calls the Chronos contract to repeatedly execute the increment() function on the newly deployed SuccessCounter contract.

How It Works
Setup: The script installs the required Solidity compiler version (0.8.9).

Account Loading: It securely loads a private key from a local private_keys.txt file to sign transactions.

Contract Deployment: The SuccessCounter Solidity source code is compiled on the fly, and the contract is deployed to the Helios testnet.

Cron Creation: A transaction is sent to the main Chronos contract (0x...830) with instructions for the job:

The address of the SuccessCounter contract to call.

The ABI of the contract.

The function to call (increment).

The frequency of execution (e.g., every 50 blocks).

A deposit of HLS tokens to pay for the future gas fees of the automated calls.

Confirmation: The script waits for the transactions to be confirmed and prints the deployed contract address and the resulting Cron ID.

Prerequisites
Python 3.8+

An account on the Helios testnet with some HLS tokens to pay for gas fees.

Setup & Installation
Clone or download the script.

Create the Private Key File:
In the same directory as the script, create a file named private_keys.txt. Inside this file, paste the private key of your funded Helios testnet account.

# private_keys.txt
your_private_key_here_0123456789abcdef...

Note: The script only reads the first line of this file.

Install Dependencies:
Install the required Python packages using the requirements.txt file.

pip install -r requirements.txt

Usage
Once the setup is complete, simply run the Python script from your terminal:

python your_script_name.py

The script will output the address of your newly deployed SuccessCounter contract and the ID of the scheduled cron job. You can monitor the contract on a block explorer to see its count variable increase automatically over time.
