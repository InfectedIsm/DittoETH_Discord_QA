====Discord Q&A=====

>Q: The ERC20 tokens the VaultFacet refers to are the pegged assets in the system, right?
>
>A: "asset" should refer to the pegged asset in Vault
bridge is for the LST (reth/steth)
zeth is just an internal wrapper (like weth?) 
also to note is that most of these are "virtual", it only actually mints when you withdraw to save on gas 
at least that is why in Vault that's the only place it does mint/burn
and only transfer is in Bridge (whether eth/lst), acts like escrow vs a swap



>Q: The assets that are used are RWA right?
>
>A: ye that's the intention, just starting with USD though, have support for adding more depending on a price feed - can update with diamond if need a change. some of this I wrote in known issues issue (there's a lot). All collateralization is lst eth only. 


>Q: When are users actually getting their asset minted, I mean for when more assets get added since it is only USD for now
>
>A: you only get the asset minted when you withdraw (vaultfacet), so it's all virtual/in storage accounting (I believe like uniswap v4?)
only can create new assets with OwnerFacet createMarket (dao/admin only) so not anyone can create a new type 


>Q: Idgi Who are shorters?
>
>A: shorters are those that create a debt position in system. let me know if there's a better way to explain this - they are like CDP creators in Maker, and Liquity Trove creators? the shorter would owe the stablecoin
and the system uses an orderbook so instead of a single user creating a CDP where they have a debt position and the stablecoin, it's split into the buyer and the shorter position
buyer wants the stablecoin (providing eth), and the shorter gets into a short position (debt of stablecoin, also providing eth), but they get the yield of the collateral 
https://dittoeth.com/glossary#shortrecord
https://dittoeth.com/technical/ob#short-orders


>Q: Will dittoETH have a cUSD/ETH price feed? Presumably cUSD liquidity will be available elsewhere (uniswap etc) and its value will fluctuate there. Or do you just rely on arbitrage bots to notice fluctuations and get the cUSD price back on track?
Then you have the issue of bootstrapping cUSD liquidity on third party mkts.... You don't want there to be a pool but it be a pool with thin liquidity bc then you're subject to price manipulation and risk losing your peg
>
>A: Great question. There won't need to be a price feed, because the trading activity on the order book will act as the price feed for other platforms. Bootstrapping on external markets wont be as necessary because I am not creating a CDP. Most liquidity can stay in-house in the order book. I will make a blog post about this is a couple weeks or so.
If there are differences between an AMM's price versus the one in DittoETH's order book, then it provides for an arbitrage opportunity. The order book should have prices near the desired asset it is trying to track.


>Q: And what about deposited ETH?
>
>A: like for depositing normal eth, it also gets turned into zeth (by calling the deposit for one of the LSTs)
and to be clear it's not turned into "zeth" as an erc20 until it's withdrawn. in the docs I try to say it's "virtual" via the ethEscrowed variable
so 
rETH -> ethEscrowed (virtual zeth)
stETH -> ethEscrowed
eth -> either rETH/stETH -> ethEscrowed 


>Q: can you explain the purpose of the zethVault mapping? How come there can be multiple addresses of zeth?
>
>A: i wrote this in this section https://dittoeth.com/technical/concepts#zeth
good q! to start, there will only be 1 vault to start with. 
but in the future i want the ability to have different vaults (and different collaterals for each, which means they might have different levels of risk/yield), if true it might be better to separate the risk between vaults. so the thought is either i add the ability to convert between them if diff vaults have diff values/collateral ratios, or i leave the simplier thing of just having different zeth's (which is annoying)


>Q: So the idea is that zethVault will keep track which AppStorage.vault is using a given collateral?
>
>A: right each zeth will have their own vault but if you look at https://github.com/Cyfrin/2023-09-ditto/blob/22b189fffa00101a94b3c717bdd84e87f1f259c5/contracts/facets/VaultFacet.sol#L39-L44
i also added a check so that the first/base/normal zeth is hardcoded to save on gas because then you don't have to look up which vault it is (also Vault.CARBON is just 1)
so it's just a future facing thing, there's potential to have multiple vaults (and a diff zeth for one), if it feels like the collateral for that vault is different enough to require making another zeth 


>Q: For, eg, createBid, one of the parameters is orderHintArray which is to facilitate finding an Id to reuse without looping through 65000 of them. But a user wouldn't be able to supply that hint array parameter when creating their bid, so where does it come from?
>
>A: "hint" is basically something that will have to come from a frontend/ui or simply calculated
basically the hint would be found off-chain and added to the call automatically
the issue with a ob on-chain without this is that you can't simply loop from the first order until you find the right spot since it could take a long time, so instead you provide the hint and just verify it's the right spot
https://dittoeth.com/technical/ob#order-hints
and to be the clear, the user isn't expected to do this at all, the ui should do it for them (in the code i have some helper view functions to do this as well, though maybe also good to write the js code so don't need to call those) 


>Q:So there is a question I asked Ditto in private but since it is helpful in understanding the codebase better i pasted it here for Ditto to answer: so I was looking at the BridgeRouterFacet.sol and noticed that deposit functions do not mint zETH. I read the docs about virtual assets and accounting but then i went to VaultFacet.sol and saw that depositZETH and withdrawZETH require that user has ZETH. So the question is how is the zETH minting handeled. In tests i noticed that it is just minted by the diamond contract straight to the user. Could it be the case that i missed some steps that seperate depositing in BridgeRouterFacet.sol and depositing in VaultFacet.sol? I follow system lifecycle from docs to figure out the path / flow of transactions (steps in system).
>
>A: yep! so
withdrawZETH actually mints it
depositZETH burns it
deposit/withdraw just sets ethEscrowed
so you need to start with either eth/steth/reth
depositing any of those 3 will add to your ethEscrowed
or if you already have zeth, it will accept it but it just burns it to give you ethEscrowed
so you don't need to depositZETH after a deposit from the "bridge" 
the "zeth" as an erc20 token is just there if people want to get out and they don't want to leave with steth/reth/eth - so it's not really intended to be used and could even be entirely removed as a token since it's just virtual in the system anyway - maybe virtual eth is a better name
