# Chronos Smart Contract Scheduler

This script demonstrates how to programmatically deploy a simple smart contract and then schedule a recurring function call (a "cron job") using the Chronos protocol on the Helios testnet.

The script performs two main actions:
1.  **Deploys `SuccessCounter.sol`**: A simple contract that has a public `count` variable and an `increment()` function.
2.  **Schedules a Cron Job**: It calls the Chronos contract to repeatedly execute the `increment()` function on the newly deployed `SuccessCounter` contract.

---

## How It Works

1.  **Setup**: The script installs the required Solidity compiler version (`0.8.9`).
2.  **Account Loading**: It securely loads a private key from a local `private_keys.txt` file to sign transactions.
3.  **Contract Deployment**: The `SuccessCounter` Solidity source code is compiled on the fly, and the contract is deployed to the Helios testnet.
4.  **Cron Creation**: A transaction is sent to the main Chronos contract (`0x...830`) with instructions for the job:
    * The address of the `SuccessCounter` contract to call.
    * The ABI of the contract.
    * The function to call (`increment`).
    * The frequency of execution (e.g., every 50 blocks).
    * A deposit of HLS tokens to pay for the future gas fees of the automated calls.
5.  **Confirmation**: The script waits for the transactions to be confirmed and prints the deployed contract address and the resulting Cron ID.

---

## Prerequisites

* Python 3.8+
* An account on the Helios testnet with some HLS tokens to pay for gas fees.

---

## Setup & Installation

1.  **Clone or download the script.**

2.  **Create the Private Key File**:
    In the same directory as the script, create a file named `private_keys.txt`. Inside this file, paste the private key of your funded Helios testnet account.

    ```
    # private_keys.txt
    0xblablabla
    ```
    **Note**: The script only reads the first line of this file.

3.  **Install Dependencies**:
    Install the required Python packages using the `requirements.txt` file.

    ```bash
    pip install -r requirements.txt
    ```

---

## Usage

Once the setup is complete, simply run the Python script from your terminal:

```bash
python chronos-deploy.py
