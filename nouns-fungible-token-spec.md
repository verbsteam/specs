# Nouns Fungible Token V1 spec

TODO

- deposit/mint(nounId)
   - Transfers erc721 noun to contract
   - Mints 1M $nouns, sends to caller (or optional to address)
   - Q: anyone, or only DAO?
      - Do we want a DAO-configurable flag?
- withdraw/redeem(nounId)
   - Burns 1M $nouns of the caller
   - Transfers erc721 noun with id nounId to caller (or optional to address)
   - Anyone can call this
- Contract is upgradeable by the DAO
- ERC20 stuff
   - Name: Nouns (same as the Nouns NFT) ?
   - Symbol: NOUNS (Nouns NFT symbol is NOUN) ?
   - Decimals: 18?
- interaction with fork mechanism
   - change adjustedTotalSupply to subtract $nouns in treasury divided by the exchange rate, add explanation to why
   - Don’t allow adding $nouns to the erc20 whitelist
