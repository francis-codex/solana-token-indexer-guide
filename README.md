# Building a Real-Time Token Transfer Indexer on Solana Using QuickNode Streams and Functions 👨🏾‍💻

This guide walks through the process of creating a real-time token transfer indexer for SPL tokens and Metaplex NFTs on Solana, using QuickNode’s Streams and Functions. The indexer will monitor token transfers, process the data, and send enriched information to a webhook for further use.


## What We Will Do?
- Set up a Stream on QuickNode that filters SPL Token transfers and Metaplex NFT (Standard) transaction events from the blockchain.
- Implement a Function that processes and enriches the transfer data and sends the processed data to a webhook.

## What We Will Need?
- A Solana QuickNode endpoint.
- Familiarity with JavaScript/TypeScript and Solana’s ecosystem.
- A webhook URL to receive the enriched token transfer data. You can get a free webhook URL at [Webhook](https://webhook.site/#!/).
- QuickNode Streams and Functions enabled on your QuickNode account.

### Building a Token Transfer Indexer

#### Create a Stream on QuickNode
First, navigate to the [QuickNode Stream Page](https://dashboard.quicknode.com/streams/new) in the Dashboard and click "Create Stream".

Next, create a Stream with the following settings:

Chain: Solana <br>
Network: Mainnet <br>
Dataset: Programs + Logs <br>
Stream start: Latest block (you can change this based on your needs) <br>
Stream payload: Modify the Stream before streaming <br>
Reorg Handling: Leave as-is

Select the option to modify the payload before streaming.

Next, copy and paste the following code to filter the streaming data for spl and metaplex token transfers, extract key data, and generate the payload to send to your function for processing.
~~~javascript
// Stream filter configuration
const streamFilter = {
  "includeTransactions": true,
  "filterPrograms": [
    "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
    "metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s"
  ]
};

function main(stream) {
  try {
    const data = stream.data ? stream.data : stream;
    if (!data || !data.transaction) {
      return null;
    }

    // Program IDs
    const TOKEN_PROGRAM_ID = 'TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA';
    const METADATA_PROGRAM_ID = 'metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s';

    const splTokenTransfers = [];
    const nftTransfers = [];

    // Process transaction
    const transaction = data.transaction;
    const timestamp = data.blockTime ? data.blockTime * 1000 : Date.now();

    // Extract accounts involved in the transaction
    const accountKeys = transaction.message.accountKeys;

    // Process instructions
    transaction.message.instructions.forEach(instruction => {
      const programId = accountKeys[instruction.programIdIndex].pubkey;

      // SPL Token Transfer
      if (programId === TOKEN_PROGRAM_ID) {
        // Check if it's a transfer instruction (3 is the transfer instruction index)
        if (instruction.data[0] === 3) {
          const sourceIndex = instruction.accounts[0];
          const destIndex = instruction.accounts[1];
          const authorityIndex = instruction.accounts[2];

          splTokenTransfers.push({
            type: 'SPL_TOKEN',
            sender: accountKeys[sourceIndex].pubkey,
            receiver: accountKeys[destIndex].pubkey,
            authority: accountKeys[authorityIndex].pubkey,
            mint: accountKeys[instruction.accounts[3]]?.pubkey, // Token mint address
            amount: instruction.data.slice(1, 9).readBigUInt64LE(0).toString(),
            txSignature: data.signature,
            slot: data.slot,
            blockTimestamp: timestamp
          });
        }
      }
      // Metaplex NFT Transfer
      else if (programId === METADATA_PROGRAM_ID) {
        // Process NFT transfers - looking for metadata updates that indicate transfers
        const metadataAccount = accountKeys[instruction.accounts[0]].pubkey;
        const updateAuthority = accountKeys[instruction.accounts[1]]?.pubkey;

        if (instruction.data[0] === 0) { // CreateMetadata instruction
          nftTransfers.push({
            type: 'METAPLEX_NFT',
            metadataAccount,
            updateAuthority,
            mint: accountKeys[instruction.accounts[2]]?.pubkey,
            action: 'CREATE',
            txSignature: data.signature,
            slot: data.slot,
            blockTimestamp: timestamp
          });
        }
      }
    });

    // Process inner instructions if any
    if (transaction.meta?.innerInstructions) {
      transaction.meta.innerInstructions.forEach(innerSet => {
        innerSet.instructions.forEach(instruction => {
          const programId = accountKeys[instruction.programIdIndex].pubkey;
          
          if (programId === TOKEN_PROGRAM_ID) {
            // Process inner token transfers
            if (instruction.data[0] === 3) {
              const sourceIndex = instruction.accounts[0];
              const destIndex = instruction.accounts[1];
              const authorityIndex = instruction.accounts[2];

              splTokenTransfers.push({
                type: 'SPL_TOKEN_INNER',
                sender: accountKeys[sourceIndex].pubkey,
                receiver: accountKeys[destIndex].pubkey,
                authority: accountKeys[authorityIndex].pubkey,
                mint: accountKeys[instruction.accounts[3]]?.pubkey,
                amount: instruction.data.slice(1, 9).readBigUInt64LE(0).toString(),
                txSignature: data.signature,
                slot: data.slot,
                blockTimestamp: timestamp
              });
            }
          }
        });
      });
    }

    if (!splTokenTransfers.length && !nftTransfers.length) {
      return null;
    }

    return {
      splToken: splTokenTransfers,
      nft: nftTransfers
    };

  } catch (e) {
    console.error('Error in main function:', e);
    return { error: e.message };
  }
}
~~~
#### Test the Stream filter
Click the Run test button to test your filter against a single block of data. Once the test is complete, you will see a sample of the data payload that the Stream will generate.

Click the Next button and then choose "Functions" for your Stream destination. Then, in the Function dropdown, choose the "Create a new Function" option.

Implement the Function
In the Functions settings page, under "Select a namespace", choose "Create a new namespace" and input a name (i.e. Solana-token-indexer), then click "Create namespace". Next, give your Function a name (i.e., Solana-token-indexer), and then click Next (you can leave all the other settings as the defaults). In the next step, paste the following code to implement the Function that retrieves token metadata via RPC. Make sure to update the values for YOUR_QUICKNODE_SOLANA_ENDPOINT, YOUR_WEBHOOK_URL, and YOUR_IPFS_GATEWAY with your real values.
~~~javascript

~~~
