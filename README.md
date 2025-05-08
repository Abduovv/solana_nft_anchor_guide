# A Guide to NFT Protocols on Solana Using Anchor

NFTs on Solana are implemented on top of the SPL Token program and Metaplex’s Token Metadata program. An NFT is simply an SPL token with **supply = 1** whose mint is frozen (no further tokens can be minted). All identifying information (name, image, attributes, royalties, etc.) is stored in a separate **metadata account** managed by the Metaplex Token Metadata program. In practice, minting an NFT involves these steps: create a new SPL mint, mint exactly one token to a user account, freeze the mint, and then initialize a metadata PDA for that mint. The metadata PDA (a *Program Derived Address*) is seeded by the mint’s public key, the token metadata program ID, and the literal `"metadata"`. The on-chain metadata account stores fields like **name**, **symbol**, **URI**, **seller fee**, **creators**, etc., and its `uri` field points to an off-chain JSON (typically on Arweave/IPFS) containing additional fields like description, image link, and attributes. In short, Solana NFTs leverage the standard Token program for on-chain token logic and the Metaplex protocol for rich metadata.

* **SPL Token for NFT:** On Solana, an NFT uses the SPL Token program. You create a new mint with 0 decimals and mint exactly 1 token, then freeze the mint (to prevent more minting).
* **Metaplex Metadata:** A separate on-chain Metaplex program attaches the “extra” data. It uses a PDA derived from the mint to store fields like name, symbol, and a URI that points to external JSON.
* **Master Edition:** For NFTs, a *Master Edition* PDA can be created to mark a token as a non-fungible asset. The Master Edition account (another PDA) “locks” the mint and freeze authorities, effectively proving that no one can mint more without going through the metadata program. This also enables printing limited editions of the original NFT.

## Creating and Minting NFTs Using Anchor

Using Anchor, you can write a Rust on-chain program that handles the SPL token and metadata CPIs. The typical approach is:

1. **Initialize the Mint:** Use `anchor_spl::token::initialize_mint` to create a new mint with 0 decimals, and set the mint authority and freeze authority (often to the same signer).
2. **Create the Token Account:** Create an associated token account for the recipient where the 1 NFT token will go.
3. **Mint 1 Token:** Use `anchor_spl::token::mint_to` to mint exactly 1 token of the new mint into the token account.
4. **Freeze the Mint (optional):** After minting, you can use `freeze_mint` or `freeze_account` to disable further minting (often done by setting the freeze authority when initializing the mint).
5. **Create Metadata PDA:** Finally, invoke the Metaplex Token Metadata program via CPI to create the metadata account for this mint.

Below is a simplified Anchor instruction showing how to mint an NFT and set its metadata. It assumes an `InitNft` context with the necessary accounts (new mint, token account, payer, Metaplex program, etc.):

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, Mint, TokenAccount};

declare_id!("YourProgramID...");

#[derive(Accounts)]
pub struct InitNft<'info> {
    /// The mint for the NFT (will be created as a PDA by this program).
    #[account(init, payer = authority, space = 8 + Mint::LEN)]
    pub mint: Account<'info, Mint>,
    /// The token account that holds the minted NFT.
    #[account(init, payer = authority,
        associated_token::mint = mint,
        associated_token::authority = authority)]
    pub token_account: Account<'info, TokenAccount>,
    /// The fee payer and authority for minting.
    #[account(mut)]
    pub authority: Signer<'info>,
    /// Standard programs.
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
    /// Metaplex Token Metadata Program (address is a constant Metaplex ID).
    pub token_metadata_program: Program<'info, anchor_spl::metadata::Metadata>,
    /// The PDA to hold metadata (initialized by Metaplex CPI; use `UncheckedAccount`).
    #[account(mut)]
    pub metadata: UncheckedAccount<'info>,
}

pub fn init_nft(ctx: Context<InitNft>, name: String, symbol: String, uri: String) -> Result<()> {
    // 1. Initialize the mint with 0 decimals, authority sets to `authority`.
    token::initialize_mint(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            token::InitializeMint {
                mint: ctx.accounts.mint.to_account_info(),
                rent: ctx.accounts.rent.to_account_info(),
            },
        ),
        0, // 0 decimals
        ctx.accounts.authority.key,
        Some(ctx.accounts.authority.key),
    )?;

    // 2. Mint 1 token to the token_account.
    token::mint_to(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            token::MintTo {
                mint: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
        ),
        1, // amount = 1 NFT
    )?;

    // (Optional) 3. Freeze the mint so no further tokens can be minted.
    // token::set_authority(...) to remove mint_authority, if desired.

    // 4. Create the metadata account via Metaplex CPI.
    let metadata_accounts = anchor_spl::token::CreateMetadataAccounts {
        metadata: ctx.accounts.metadata.to_account_info(),
        mint: ctx.accounts.mint.to_account_info(),
        mint_authority: ctx.accounts.authority.to_account_info(),
        payer: ctx.accounts.authority.to_account_info(),
        update_authority: ctx.accounts.authority.to_account_info(),
        system_program: ctx.accounts.system_program.to_account_info(),
        rent: ctx.accounts.rent.to_account_info(),
        token_metadata_program: ctx.accounts.token_metadata_program.to_account_info(),
    };
    let cpi_ctx = CpiContext::new(ctx.accounts.token_metadata_program.to_account_info(), metadata_accounts);

    // Data for the NFT: name, symbol, uri, seller fee, creators, etc.
    let data = anchor_spl::metadata::Data {
        name: name.clone(),
        symbol: symbol.clone(),
        uri: uri.clone(),
        seller_fee_basis_points: 0,
        creators: None,
    };
    // Invoke the Metaplex instruction to create metadata (CPI).
    anchor_spl::metadata::create_metadata_accounts(
        cpi_ctx,
        data,
        true,  // is_mutable
        false, // update_authority_is_signer
    )?;

    Ok(())
}
```

In this snippet:

* We **initialize the mint** with `token::initialize_mint`, setting decimals to 0 and both mint & freeze authority to our `authority` signer.
* We **mint exactly 1 token** to the newly created token account.
* We then prepare a CPI to Metaplex’s `create_metadata_accounts` via `anchor_spl::metadata`. The metadata PDA (`metadata`) is an unchecked account that the Metaplex program will create (via its PDA seeds). We pass in the NFT’s name, symbol, and URI (typically pointing to an Arweave/IPFS JSON) in the `Data` struct.
* **Anchor simplifies** the flow: because we used the `init` attribute on the mint, Anchor auto-creates and signs the mint account for us, so we don’t explicitly call `create_mint`. The CPI `create_metadata_accounts` call will do the rest.

## Metaplex Token Metadata Program (Deep Dive)

The Metaplex Token Metadata program is responsible for all NFT metadata logic. It uses **Program Derived Addresses (PDAs)** tied to each mint. The **metadata account** PDA is derived from the seeds: the literal `"metadata"`, the program ID, and the mint’s public key. In code, you could derive it as:

```rust
let (metadata_pda, _) = Pubkey::find_program_address(
    &[
        b"metadata",
        mpl_token_metadata::ID.as_ref(),
        mint_pubkey.as_ref(),
    ],
    &mpl_token_metadata::ID,
);
```

The metadata PDA’s account data (a `Metadata` struct) contains fields including an account discriminator (“Key”), the *update authority* pubkey, the mint pubkey it’s associated with, the NFT’s on-chain **name**, **symbol**, **URI**, **seller\_fee\_basis\_points**, **primary\_sale\_happened**, **is\_mutable**, and optional **creators** array, among others. Crucially, the `uri` field in this account holds a link to an off-chain JSON that contains the image link, description, traits, etc.

To make a token non-fungible, you also typically create a **Master Edition** PDA for the mint. The Master Edition account (seeded by `"edition"` or similar) is a special PDA that, when initialized, transfers the mint and freeze authority into the metadata program. This “locks” the mint so that only one total supply can exist, and acts as proof of non-fungibility. Once a master edition exists, you can even print limited editions (child NFTs) from it using the `mint_new_edition_from_master_edition` instruction.

The Token Metadata program exposes key instructions (all callable via CPI from Anchor):

* `create_metadata_accounts` (or v3 versions) to create the metadata PDA.
* `update_metadata_account` to change fields if the NFT is mutable.
* `create_master_edition` to turn a mint into an NFT and set up editions.
* `mint_new_edition_from_master_edition_via_token` to mint copies from a master edition.

These CPIs manage the account layouts and enforce NFT rules. For example, before `create_master_edition` can succeed, the program checks the mint has supply = 1 and that no additional tokens have been minted. It also transfers the mint authority into the Master Edition PDA.

In practice, to interact with these, your Anchor program CPI to `create_metadata_accounts_v3` (for example) might look like this (simplified):

```rust
let ctx = CpiContext::new(
    ctx.accounts.token_metadata_program.to_account_info(),
    CreateMetadataAccountsV3 {
        metadata: ctx.accounts.metadata.to_account_info(),
        mint: ctx.accounts.mint.to_account_info(),
        mint_authority: ctx.accounts.mint_authority.to_account_info(),
        payer: ctx.accounts.payer.to_account_info(),
        update_authority: ctx.accounts.update_authority.to_account_info(),
        system_program: ctx.accounts.system_program.to_account_info(),
        rent: ctx.accounts.rent.to_account_info(),
    },
    &[&[b"mint", &[ctx.bumps.mint]]], // if mint is a PDA
);
anchor_spl::mpl_token_metadata::create_metadata_accounts_v3(
    ctx,
    DataV2 {
        name: name.clone(),
        symbol: symbol.clone(),
        uri: uri.clone(),
        seller_fee_basis_points: 0,
        creators: None,
        collection: None,
        uses: None,
    },
    false, // is_mutable
    true,  // update authority is signer
    None,  // no collection
)?;
```

This CPI creates the metadata account on-chain. Anchor handles signing (with `new_with_signer`) if any PDAs are involved. The end result is an on-chain metadata account (PDA) populated with the NFT’s name, symbol, URI, etc.

**Account Summary:** The metadata PDA stores the core NFT data. The Master Edition PDA (if created) enforces the 1-supply rule and can hold edition info. Holders of the actual token use normal SPL Token accounts; metadata transfers do not involve the metadata program – token transfers are done via the SPL Token program (unchanged). Note that marketplaces read the metadata PDA to show images/traits, so ensuring the URI points to a correct JSON (see below) is crucial.

## Custom NFT Protocol (Without Metaplex)

You can also build your own NFT logic without using Metaplex. Two common approaches are:

* **Using Token-2022 Metadata Extensions:** Solana’s new Token-2022 program supports a built-in *metadata extension*. With Anchor v0.30+, you can initialize on-chain metadata directly via `token_metadata_initialize`. For example, Anchor provides a CPI like `token_metadata_initialize(cpi_ctx, name, symbol, uri)?;` which embeds the metadata (name, symbol, uri) in the mint’s extensions. This avoids a separate metadata program altogether. The CPI call might look like:

  ```rust
  let cpi_ctx = CpiContext::new(
      ctx.accounts.token_program.to_account_info(),
      TokenMetadataInitialize {
          mint: ctx.accounts.mint.to_account_info(),
          metadata: ctx.accounts.mint.to_account_info(), // using the mint address itself for metadata PDA
      },
  );
  anchor_spl::token::token_metadata_initialize(
      cpi_ctx,
      name.clone(),
      symbol.clone(),
      uri.clone(),
  )?;
  // The above call stores `name`, `symbol`, `uri` on-chain in the Token-2022 metadata extension:contentReference[oaicite:23]{index=23}.
  ```

* **Defining Custom Metadata Accounts:** Alternatively, you can create your own Anchor account struct to hold NFT metadata. For example, define:

  ```rust
  #[account]
  pub struct MyNftMetadata {
      pub owner: Pubkey,      // owner of this NFT
      pub name: String,       // NFT name
      pub uri: String,        // URI to off-chain JSON
      // ... any other fields ...
  }
  ```

  Then write an instruction to initialize and mint the NFT by creating this account (often as a PDA or via `init`). For instance:

  ```rust
  #[derive(Accounts)]
  pub struct CreateMyNft<'info> {
      #[account(init, payer = creator, space = 8 + 32 + 4 + 100 + 4 + 200)]
      pub nft_account: Account<'info, MyNftMetadata>,
      #[account(mut)]
      pub creator: Signer<'info>,
      pub system_program: Program<'info, System>,
  }
  pub fn create_my_nft(ctx: Context<CreateMyNft>, name: String, uri: String) -> Result<()> {
      let nft = &mut ctx.accounts.nft_account;
      nft.owner = *ctx.accounts.creator.key;  // set initial owner
      nft.name = name;
      nft.uri = uri;
      Ok(())
  }
  ```

  In this design, the “minting” is simply creating a new `MyNftMetadata` account on-chain. To **verify ownership** when transferring, Anchor’s account constraints help. For example:

  ```rust
  #[derive(Accounts)]
  pub struct TransferMyNft<'info> {
      #[account(mut, has_one = owner)]
      pub nft_account: Account<'info, MyNftMetadata>,
      pub owner: Signer<'info>,       // current owner must sign
      /// CHECK: new owner can be any pubkey
      pub new_owner: UncheckedAccount<'info>,
  }
  pub fn transfer_my_nft(ctx: Context<TransferMyNft>) -> Result<()> {
      let nft = &mut ctx.accounts.nft_account;
      nft.owner = *ctx.accounts.new_owner.key; // transfer ownership
      Ok(())
  }
  ```

  Here, `#[account(has_one = owner)]` ensures that the `owner` field in `MyNftMetadata` matches the signer’s pubkey, preventing unauthorized changes. This pattern shows how to implement NFT logic entirely within your program, without relying on Metaplex. (The downside is you forfeit marketplace compatibility unless you replicate Metaplex conventions.)

In both custom approaches, the essential idea is the same: create a unique identifier (e.g., a mint or PDA) for each NFT, associate metadata (either via the token’s own extension or a program account), and enforce a single owner. The Metaplex standard is convenient and interoperable, but custom methods give you more control if you need it.

# Auditing Guide for Solana Anchor NFT Protocols

## 1. Improper Signer/Ownership Checks

**Missing Signer or Authority Constraints:** When an instruction fails to enforce the correct signer or owner checks, unauthorized users can invoke sensitive actions. For example, a transfer or configuration update may not verify the NFT owner. In Anchor, omitting a `Signer` or `has_one` constraint can let attackers pass arbitrary accounts.

```rust
// Vulnerable: no Signer/ownership check on `authority` 
#[derive(Accounts)] 
pub struct UpdateSale<'info> { 
    #[account(mut)] 
    pub sale: Account<'info, Sale>,           // no constraint on sale.owner 
    /// The authority is not required to be the signer or sale.owner 
    pub authority: UncheckedAccount<'info>,   
    pub token_program: Program<'info, Token>, 
} 
 
pub fn update_sale(ctx: Context<UpdateSale>, new_price: u64) -> Result<()> { 
    // Without a signer or has_one, anyone could call this 
    ctx.accounts.sale.price = new_price; 
    Ok(()) 
} 
```

```rust
// Fixed: enforce that `authority` signs and matches sale.owner 
#[derive(Accounts)] 
pub struct UpdateSale<'info> { 
    #[account(mut, has_one = owner)] 
    pub sale: Account<'info, Sale>, 
    pub owner: Signer<'info>,                 // must sign and match sale.owner 
    pub token_program: Program<'info, Token>, 
} 
 
pub fn update_sale(ctx: Context<UpdateSale>, new_price: u64) -> Result<()> { 
    ctx.accounts.sale.price = new_price; 
    Ok(()) 
} 
```

*Impact:* An attacker could update or transfer an NFT without permission, draining funds or stealing NFTs. (Always require `Signer<'info>` and `has_one` to validate authority.)

## 2. UncheckedAccount Re-initialization (Candy Machine v2)

**Unchecked Account Allows Account Hijack:** Using `UncheckedAccount<'info>` (Anchor’s raw `AccountInfo`) without constraints can let attackers reuse and reinitialize an already-initialized PDA. In Candy Machine v2, the `candy_machine` account was marked unchecked and mutable, enabling a **re-initialization attack**. An attacker supplied a pre-existing candy machine account with their own wallet as authority.

```rust
// Vulnerable: candy_machine is UncheckedAccount, no ownership check 
#[derive(Accounts)] 
pub struct MintNft<'info> { 
    #[account(mut)] 
    pub candy_machine: UncheckedAccount<'info>, // unchecked! 
    // ... other accounts omitted 
    pub user: Signer<'info>, 
} 
 
pub fn mint_nft(ctx: Context<MintNft>) -> Result<()> { 
    // Since candy_machine is unchecked, the attacker could use an existing PDA  
    // and hijack the mint by supplying a malicious candy_machine account. 
    Ok(()) 
} 
```

```rust
// Fixed: use Account<> and proper constraints 
#[derive(Accounts)] 
pub struct MintNft<'info> { 
    #[account(mut, has_one = authority)] 
    pub candy_machine: Account<'info, CandyMachine>, // validated type 
    pub authority: Signer<'info>,                    // must be candy_machine.update_authority 
    // ... other accounts 
} 
 
pub fn mint_nft(ctx: Context<MintNft>) -> Result<()> { 
    // Now anchor checks candy_machine.owner and authority, preventing misuse. 
    Ok(()) 
} 
```

*Impact:* Attackers can hijack NFT launches and steal sale proceeds. In practice, SolShield reported that this bug could let an attacker “change the wallet account to their own,” mint unlimited NFTs, or steal funds.

## 3. Unchecked Initialization (Candy Machine v1)

**Missing “Zero” Check on Init:** If an initializer instruction does not verify that an account is unused, attackers can reuse (reinitialize) it. Candy Machine v1 suffered this: the program did not check if the candy machine config was already initialized. Attackers injected pre-initialized accounts, changing authorities.

```rust
// Vulnerable: no check that account is uninitialized 
#[derive(Accounts)] 
pub struct InitializeCandyMachine<'info> { 
    #[account(init, payer = user, space = 8 + CONFIG_SIZE)] 
    pub config: Account<'info, CandyMachineConfig>, 
    pub user: Signer<'info>, 
} 
pub fn initialize_candy_machine(ctx: Context<InitializeCandyMachine>, data: ConfigData) -> Result<()> { 
    // If 'config' is already initialized, anchor will still allow this 
    // and overwrite fields without error, hijacking the launch. 
    ctx.accounts.config.update_authority = ctx.accounts.user.key(); 
    Ok(()) 
} 
```

```rust
// Fixed: require account to be zeroed (uninitialized) 
#[derive(Accounts)] 
pub struct InitializeCandyMachine<'info> { 
    #[account(init, payer = user, space = 8 + CONFIG_SIZE, zero)] 
    pub config: Account<'info, CandyMachineConfig>, 
    pub user: Signer<'info>, 
} 
pub fn initialize_candy_machine(ctx: Context<InitializeCandyMachine>, data: ConfigData) -> Result<()> { 
    ctx.accounts.config.update_authority = ctx.accounts.user.key(); 
    Ok(()) 
} 
```

*Impact:* An attacker draining \$3M+ across \~3,500 collections. By reusing a config account, they could “populate their own address as the authority” and siphon all mint proceeds. The fix (as in Metaplex’s patch) is to use `#[account(zero)]` so Anchor rejects already-initialized accounts.

## 4. Insecure PDA/Bump Usage (Auction House)

**Improper PDA Bump Handling:** Passing a PDA bump as a user-provided argument (or not verifying it) can open up misuse. Metaplex Auction House did this for the seller trade state. An attacker could choose a bump of `0` and include extra lamports to keep the account alive. The program never checked that the bump was canonical, so `bump=0` (valid for \~50% of PDAs) was accepted.

```rust
// Vulnerable: Accepts any bump in the Sell instruction 
#[derive(Accounts)] 
#[instruction(seller_bump: u8)] 
pub struct Sell<'info> { 
    #[account(init_if_needed, 
        seeds = [b"auction_house", auction_house.key().as_ref(), seller.key().as_ref()], 
        bump = seller_bump, 
        payer = seller)] 
    pub seller_trade_state: Account<'info, TradeState>, 
    pub seller: Signer<'info>, 
    pub auction_house: Account<'info, AuctionHouse>, 
    // ... 
} 
pub fn sell(ctx: Context<Sell>, price: u64) -> Result<()> { 
    // No check that seller_bump is correct; bump=0 could be stored. 
    Ok(()) 
} 
```

```rust
// Fixed: Remove bump argument, let Anchor derive it 
#[derive(Accounts)] 
pub struct Sell<'info> { 
    #[account(init_if_needed, 
        seeds = [b"auction_house", auction_house.key().as_ref(), seller.key().as_ref()], 
        bump)]                    // Anchor ensures correct bump 
    pub seller_trade_state: Account<'info, TradeState>, 
    pub seller: Signer<'info>, 
    pub auction_house: Account<'info, AuctionHouse>, 
    // ... 
} 
pub fn sell(ctx: Context<Sell>, price: u64) -> Result<()> { 
    Ok(()) 
} 
```

*Impact:* A seller listing an NFT with `bump=0` could unwittingly leave a “persistent” sell order. An attacker could later use that old trade state to buy the NFT at the prior (likely low) price indefinitely. In Andy Kutruff’s post, this flaw let a victim’s old listing remain active, so the attacker “now has control” of that sale.

## 5. Insecure Account Closure (“Revival” Attack)

**Account Not Properly Closed:** If a program manually zeros an account’s lamports/data (instead of using Anchor’s `close`), an attacker can prevent closure. In Auction House’s `execute_sale`, the code set `lamports = 0` but a crafty buyer could send extra lamports to the trade state account in the same transaction. The garbage collector would then see a non-zero balance, leaving the account open.

```rust
// Vulnerable: manual close with memzero (no `close =`) 
#[derive(Accounts)] 
pub struct ExecuteSale<'info> { 
    #[account(mut)] 
    pub seller_trade_state: AccountInfo<'info>, 
    // ... 
} 
pub fn execute_sale(ctx: Context<ExecuteSale>) -> Result<()> { 
    // Attempt to close seller_trade_state manually: 
    **ctx.accounts.seller_trade_state.lamports.borrow_mut() = 0; 
    sol_memset(&mut ctx.accounts.seller_trade_state.try_borrow_mut_data()?, 0, ctx.accounts.seller_trade_state.data_len()); 
    Ok(()) 
} 
```

```rust
// Fixed: use `close` to safely transfer lamports 
#[derive(Accounts)] 
pub struct ExecuteSale<'info> { 
    #[account(mut, close = seller)] 
    pub seller_trade_state: Account<'info, TradeState>, 
    #[account(mut)] 
    pub seller: AccountInfo<'info>,      // receives lamports 
    // ... 
} 
pub fn execute_sale(ctx: Context<ExecuteSale>) -> Result<()> { 
    // Anchor will automatically close seller_trade_state and send lamports to seller. 
    Ok(()) 
} 
```

*Impact:* As described by Andy Kutruff, combining `execute_sale` with a simple SOL transfer can keep the trade state account alive. The attacker (buyer) “controls whether the seller\_trade\_state account is still open after sale execution”. This leaves a perpetual sell order on the books.

## 6. Misuse of Metaplex Metadata (Collection Tampering)

**Unauthorized Collection Setting:** The Metaplex token metadata program allows setting a certified collection, but improper checks can be abused. SolShield found a flaw where anyone with delegate privileges could set the NFT’s collection to an arbitrary mint. Anchors-based CPI calls to `set_and_verify_collection` must enforce that the update authority or approved delegate is signing.

```rust
// Vulnerable: CPI without verifying update authority 
#[derive(Accounts)] 
pub struct SetCollection<'info> { 
    /// NFT metadata to modify 
    #[account(mut)] 
    pub metadata: UncheckedAccount<'info>, 
    #[account(mut)] 
    pub collection_mint: UncheckedAccount<'info>, 
    pub collection_metadata: UncheckedAccount<'info>, 
    pub authority: Signer<'info>, 
    pub token_metadata_program: Program<'info, TokenMetadata>, 
} 
pub fn set_collection(ctx: Context<SetCollection>) -> Result<()> { 
    // Missing check: is `authority` actually update authority or delegate? 
    set_and_verify_collection(...); // hypothetical CPI 
    Ok(()) 
} 
```

```rust
// Fixed: ensure authority is verified (Anchor will ensure signer and PDA seeds) 
#[derive(Accounts)] 
pub struct SetCollection<'info> { 
    #[account(mut, has_one = update_authority)] 
    pub metadata: Account<'info, Metadata>, 
    #[account(mut)] 
    pub collection_mint: Account<'info, Mint>, 
    pub collection_metadata: UncheckedAccount<'info>, 
    pub update_authority: Signer<'info>,  // must match metadata.update_authority 
    pub token_metadata_program: Program<'info, TokenMetadata>, 
} 
pub fn set_collection(ctx: Context<SetCollection>) -> Result<()> { 
    // Anchor’s constraints ensure only the real update authority can sign. 
    set_and_verify_collection(...); 
    Ok(()) 
} 
```

*Impact:* Without proper authority checks, an attacker could “set the on-chain certified collection of the metadata account without proper authorization”. This lets them group random NFTs into a fake collection, tricking marketplaces and defrauding users.

## 7. Unlimited Minting (Missing Mint Control)

**No Supply Cap or Fee Check:** If a mint instruction lacks checks on supply or payment, anyone can mint unlimited NFTs. For example, failing to enforce a max supply or require payment allows free or infinite token issuance.

```rust
// Vulnerable: no check on max supply or payment 
pub fn mint_nft(ctx: Context<MintNFT>) -> Result<()> { 
    let cpi_accounts = MintTo { 
        mint: ctx.accounts.mint.to_account_info(), 
        to: ctx.accounts.recipient.to_account_info(), 
        authority: ctx.accounts.mint_authority.to_account_info(), 
    }; 
    // Minting 1 token, but no cap enforced 
    token::mint_to(ctx.accounts.token_program.to_account_info(), cpi_accounts, 1)?; 
    Ok(()) 
} 
```

```rust
// Fixed: enforce supply limit or require lamports 
pub fn mint_nft(ctx: Context<MintNFT>, price: u64) -> Result<()> { 
    // Enforce payment and max supply 
    require!(ctx.accounts.collection.total_minted < ctx.accounts.collection.max_supply, CustomError::MaxSupplyReached); 
    // Transfer price, mint token 
    let cpi_accounts = MintTo { /* ... */ }; 
    token::mint_to(ctx.accounts.token_program.to_account_info(), cpi_accounts, 1)?; 
    // Update supply count 
    ctx.accounts.collection.total_minted += 1; 
    Ok(()) 
} 
```

*Impact:* Without checks, the contract could mint an unlimited number of NFTs or allow free claims. This destroys scarcity and value, as seen in NFT drops where bots or attackers mint out collections instantly.

## 8. Off-Chain Metadata & URI Risks

**Trusting External Metadata:** NFTs on Solana typically point to off-chain JSON and media (e.g. IPFS/HTTP). This external content can be altered or deleted without on-chain notice. A program that assumes metadata is immutable or secure may be vulnerable to “metadata tampering.”

*Mitigation:* Wherever possible, store hashes (of JSON/image) on-chain or use decentralized storage (Arweave/IPFS with pinned content). Avoid logic that depends on the contents of off-chain URIs, since a malicious metadata update can redirect users to phishing links or inappropriate content. (This is a **protocol-level** issue rather than Anchor code – always design for content mutability.)

## Real-World Exploits & Postmortems

* **Candy Machine v1 (Jan 2022):** A re-initialization bug drained \~3,500 projects (∼1,000 SOL each). The attacker reused initialized config accounts and redirected mint proceeds.
* **Candy Machine v2 (Dec 2021):** A similar flaw allowed unlimited minting by reusing the config account. Metaplex quickly patched it (changing an Anchor attribute).
* **Metaplex Auction House (Oct 2022):** The “persistent sale” exploit let an attacker buy NFTs at a previous (low) price indefinitely. This combined a rent-transfer trick with unchecked bump usage.
* **Token Metadata Collection Flaw (Dec 2022):** SolShield reported that one could set any NFT’s certified collection without proper authority, potentially deceiving marketplaces into accepting unrelated NFTs as part of a collection.

Each of these teaches that even well-audited protocols can fail to use Anchor’s safety features correctly.

## Best Practices for Testing & Auditing Anchor NFT Programs

* **Use Anchor Account Constraints:** Always use `Signer<'info>` for signing authorities and `has_one`, `owner`, or `address` constraints to enforce ownership. This prevents unauthorized callers.
* **Verify PDAs Properly:** Don’t accept user-supplied bumps or seeds. Let Anchor derive bump seeds automatically. For any PDA, include all expected seeds and a `bump` in the attribute to let Anchor validate it.
* **Thorough Unit/Integration Tests:** Write comprehensive tests (using Anchor’s Mocha/TypeScript or `anchor_lang` test harness) covering normal and edge cases (overflows, wrong accounts, no funds). Simulate invalid scenarios (wrong signer, reused accounts) to ensure checks work.
* **Fuzz Testing:** Use Solana fuzzers like **Trident** (open-source by Aki) or **Trdelnik** to generate random inputs and detect edge-case bugs. As Viktor Fischer notes, fuzzing is now available for Solana and should complement other practices.
* **Static Analysis:** Employ tools like **Radar** (for Anchor) or **Xray** to scan your code for known patterns (unconstrained accounts, unchecked CPIs, etc.). Custom rules can flag missing `signer` or `has_one`.
* **Code Reviews & Audits:** Peer-review every change. Formal audits by security teams can catch subtle logic errors. The Solana docs themselves provide examples of common exploits and Anchor idioms to avoid.
* **On-Chain Checks after CPIs:** After any cross-program invocation (e.g. to the token or metadata program), reload accounts and re-apply constraints if necessary. Anchor does not automatically refresh accounts post-CPI.
* **Manage Metadata Carefully:** Minimize reliance on off-chain data. Where possible, verify content hashes on-chain or use verified collections and immutable URIs. Implement updates to metadata only through controlled instructions.

