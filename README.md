1. Create Project
For create the sample near project please run `npx create-near-app`.
At that time, set the Smart Contract Language as Rust, Frontend language as React.js.
After that you can run
`npm run deploy` - for Deploy contract
`npm test` - for integration test
`npm start` - for run fontend

You can see that all is working.

2. Make Basic Project

First Step :
Make build.sh and Edit like this

**************************************************
#!/bin/bash
set -e && RUSTFLAGS='-C link-arg=-s' cargo build --target wasm32-unknown-unknown --release && mkdir -p ../out && cp target/wasm32-unknown-unknown/release/*.wasm ../out/main.wasm
**************************************************

Second Step :
Make Smart Contract files(
lib.rs : Root file
interenal.rs : Declare about Internal Methods
metadata.rs : Declare about NFT Metadata Format
mint.rs : Write Mint_NFT functions
)

``````````````````````````````````````````````````
contract
├── Cargo.lock
├── Cargo.toml
├── README.md
├── build.sh
└── src
    ├── internal.rs
    ├── lib.rs
    ├── metadata.rs
    └── mint.rs
``````````````````````````````````````````````````

3. Build Smart Contract

##lib.rs##
Modify Contract Struct
**************************************************
#[near_bindgen]
#[derive(BorshDeserialize, BorshSerialize, PanicOnDefault)]
pub struct Contract {
    //contract owner
    pub owner_id: AccountId,

    //keeps track of all the token IDs for a given account
    pub tokens_per_owner: LookupMap<AccountId, UnorderedSet<TokenId>>,

    //keeps track of the token struct for a given token ID
    pub tokens_by_id: LookupMap<TokenId, Token>,

    //keeps track of the token metadata for a given token ID
    pub token_metadata_by_id: UnorderedMap<TokenId, TokenMetadata>,

    //keeps track of the metadata for the contract
    pub metadata: LazyOption<NFTContractMetadata>,
}
**************************************************

Next, create what's called an initialization function; you can name it new. This function needs to be invoked when you first deploy the contract. It will initialize all the contract's fields that you've defined above with default values. Don't forget to add the owner_id and metadata fields as parameters to the function, so only those can be customized.

This function will default all the collections to be empty and set the owner and metadata equal to what you pass in.

**************************************************
#[init]
pub fn new(owner_id: AccountId, metadata: NFTContractMetadata) -> Self {
    //create a variable of type Self with all the fields initialized. 
    let this = Self {
        //Storage keys are simply the prefixes used for the collections. This helps avoid data collision
        tokens_per_owner: LookupMap::new(StorageKey::TokensPerOwner.try_to_vec().unwrap()),
        tokens_by_id: LookupMap::new(StorageKey::TokensById.try_to_vec().unwrap()),
        token_metadata_by_id: UnorderedMap::new(
            StorageKey::TokenMetadataById.try_to_vec().unwrap(),
        ),
        //set the owner_id field equal to the passed in owner_id. 
        owner_id,
        metadata: LazyOption::new(
            StorageKey::NFTContractMetadata.try_to_vec().unwrap(),
            Some(&metadata),
        ),
    };

    //return the Contract object
    this
}
**************************************************
Let's create a function that can initialize the contract with a set of default metadata. You can call it new_default_meta and it'll only take the owner_id as a parameter.

**************************************************
#[init]
pub fn new_default_meta(owner_id: AccountId) -> Self {
    //calls the other function "new: with some default metadata and the owner_id passed in 
    Self::new(
        owner_id,
        NFTContractMetadata {
                spec: "nft-1.0.0".to_string(),
                name: "NFT Mint Contract".to_string(),
                symbol: "NearSeed".to_string(),
                icon: None,
                base_uri: None,
                reference: None,
                reference_hash: None,
        },
    )
}
**************************************************

##metadata.rs##
Now that you've defined what information to store on the contract itself and you've defined some ways to initialize the contract, you need to define what information should go in the Token, TokenMetadata, and NFTContractMetadata data types.

**************************************************
#[derive(BorshDeserialize, BorshSerialize, Serialize, Deserialize, Clone)]
#[serde(crate = "near_sdk::serde")]
pub struct NFTContractMetadata {
    pub spec: String,              // required, essentially a version like "nft-1.0.0"
    pub name: String,              // required, ex. "Mosaics"
    pub symbol: String,            // required, ex. "MOSAIC"
    pub icon: Option<String>,      // Data URL
    pub base_uri: Option<String>, // Centralized gateway known to have reliable access to decentralized storage assets referenced by `reference` or `media` URLs
    pub reference: Option<String>, // URL to a JSON file with more info
    pub reference_hash: Option<Base64VecU8>, // Base64-encoded sha256 hash of JSON from reference field. Required if `reference` is included.
}

#[derive(BorshDeserialize, BorshSerialize, Serialize, Deserialize)]
#[serde(crate = "near_sdk::serde")]
pub struct TokenMetadata {
    pub title: Option<String>, // ex. "Arch Nemesis: Mail Carrier" or "Parcel #5055"
    pub description: Option<String>, // free-form description
    pub media: Option<String>, // URL to associated media, preferably to decentralized, content-addressed storage
    pub media_hash: Option<Base64VecU8>, // Base64-encoded sha256 hash of content referenced by the `media` field. Required if `media` is included.
    pub copies: Option<u64>, // number of copies of this set of metadata in existence when token was minted.
    pub issued_at: Option<u64>, // When token was issued or minted, Unix epoch in milliseconds
    pub expires_at: Option<u64>, // When token expires, Unix epoch in milliseconds
    pub starts_at: Option<u64>, // When token starts being valid, Unix epoch in milliseconds
    pub updated_at: Option<u64>, // When token was last updated, Unix epoch in milliseconds
    pub extra: Option<String>, // anything extra the NFT wants to store on-chain. Can be stringified JSON.
    pub reference: Option<String>, // URL to an off-chain JSON file with more info.
    pub reference_hash: Option<Base64VecU8>, // Base64-encoded sha256 hash of JSON from reference field. Required if `reference` is included.
}
#[derive(BorshDeserialize, BorshSerialize)]
pub struct Token {
    pub owner_id: AccountId,
}
**************************************************

For the Token struct, you'll just keep track of the owner for now.

**************************************************
#[derive(BorshDeserialize, BorshSerialize)]
pub struct Token {
    //owner of the token
    pub owner_id: AccountId,
}
**************************************************

Now that you've defined some of the types that were used in the previous section, let's move on and create the first view function nft_metadata. This will allow users to query for the contract's metadata as per the metadata standard.

**************************************************
pub trait NonFungibleTokenMetadata {
    //view call for returning the contract metadata
    fn nft_metadata(&self) -> NFTContractMetadata;
}

#[near_bindgen]
impl NonFungibleTokenMetadata for Contract {
    fn nft_metadata(&self) -> NFTContractMetadata {
        self.metadata.get().unwrap()
    }
}
**************************************************

##mint.rs##

Looking at these data structures, you could do the following:

 - Add the token ID into the set of tokens that the receiver owns. This will be done on the tokens_per_owner field.
 - Create a token object and map the token ID to that token object in the tokens_by_id field.
 - Map the token ID to it's metadata using the token_metadata_by_id.

**************************************************
#[near_bindgen]
impl Contract {
    #[payable]
    pub fn nft_mint(
        &mut self,
        token_id: TokenId,
        metadata: TokenMetadata,
        receiver_id: AccountId,
    ) {
        //measure the initial storage being used on the contract
        let initial_storage_usage = env::storage_usage();

        //specify the token struct that contains the owner ID 
        let token = Token {
            //set the owner ID equal to the receiver ID passed into the function
            owner_id: receiver_id,
        };

        //insert the token ID and token struct and make sure that the token doesn't exist
        assert!(
            self.tokens_by_id.insert(&token_id, &token).is_none(),
            "Token already exists"
        );

        //insert the token ID and metadata
        self.token_metadata_by_id.insert(&token_id, &metadata);

        //call the internal method for adding the token to the owner
        self.internal_add_token_to_owner(&token.owner_id, &token_id);

        //calculate the required storage which was the used - initial
        let required_storage_in_bytes = env::storage_usage() - initial_storage_usage;

        //refund any excess storage if the user attached too much. Panic if they didn't attach enough to cover the required.
        refund_deposit(required_storage_in_bytes);
    }
}
**************************************************

##internal.rs##
Now we must write internal methods for Mint_NFT

**************************************************
use crate::*;
use near_sdk::{CryptoHash};

//used to generate a unique prefix in our storage collections (this is to avoid data collisions)
pub(crate) fn hash_account_id(account_id: &AccountId) -> CryptoHash {
    //get the default hash
    let mut hash = CryptoHash::default();
    //we hash the account ID and return it
    hash.copy_from_slice(&env::sha256(account_id.as_bytes()));
    hash
}

//refund the initial deposit based on the amount of storage that was used up
pub(crate) fn refund_deposit(storage_used: u64) {
    //get how much it would cost to store the information
    let required_cost = env::storage_byte_cost() * Balance::from(storage_used);
    //get the attached deposit
    let attached_deposit = env::attached_deposit();

    //make sure that the attached deposit is greater than or equal to the required cost
    assert!(
        required_cost <= attached_deposit,
        "Must attach {} yoctoNEAR to cover storage",
        required_cost,
    );

    //get the refund amount from the attached deposit - required cost
    let refund = attached_deposit - required_cost;

    //if the refund is greater than 1 yocto NEAR, we refund the predecessor that amount
    if refund > 1 {
        Promise::new(env::predecessor_account_id()).transfer(refund);
    }
}

impl Contract {
    //add a token to the set of tokens an owner has
    pub(crate) fn internal_add_token_to_owner(
        &mut self,
        account_id: &AccountId,
        token_id: &TokenId,
    ) {
        //get the set of tokens for the given account
        let mut tokens_set = self.tokens_per_owner.get(account_id).unwrap_or_else(|| {
            //if the account doesn't have any tokens, we create a new unordered set
            UnorderedSet::new(
                StorageKey::TokenPerOwnerInner {
                    //we get a new unique prefix for the collection
                    account_id_hash: hash_account_id(&account_id),
                }
                .try_to_vec()
                .unwrap(),
            )
        });

        //we insert the token ID into the set
        tokens_set.insert(token_id);

        //we insert that set for the given account ID. 
        self.tokens_per_owner.insert(account_id, &tokens_set);
    }
} 
**************************************************

Now We Completed the Smart Contract

4. Build, Deploy Smart Contract and Mint NFT

First Step :

Build and deploy Smart Contract

``yarn build``

``near login``

To make this tutorial easier to copy/paste, we're going to set an environment variable for your account ID. In the command below, replace YOUR_ACCOUNT_NAME with the account name you just logged in with including the .testnet portion:

``export NFT_CONTRACT_ID="YOUR_ACCOUNT_NAME"``

``echo $NFT_CONTRACT_ID``

Deploy Smart Contract

``near deploy --wasmFile out/main.wasm --accountId $NFT_CONTRACT_ID``

Second Step : 

Initialize the Contract

``near call $NFT_CONTRACT_ID new_default_meta '{"owner_id": "'$NFT_CONTRACT_ID'"}' --accountId $NFT_CONTRACT_ID``

View Metadata

``near view $NFT_CONTRACT_ID nft_metadata``

Third Step : 

Mint NFT

``
near call $NFT_CONTRACT_ID nft_mint '{"token_id": "token-1", "metadata": {"title": "Seed Academy First Near NFT", "description": "This is Seed Academy First Near NFT", "media": "https://www.arweave.net/5_f4rixgIXZEEkOypQsZfj1H8mm5D29UiQT0TxEwtlY?ext=png"}, "receiver_id": "'$NFT_CONTRACT_ID'"}' --accountId $NFT_CONTRACT_ID --amount 0.1
``

View NFT

``
near view $NFT_CONTRACT_ID nft_token '{"token_id": "token-1"}'
``

You can check the NFT on your Near Wallet too.