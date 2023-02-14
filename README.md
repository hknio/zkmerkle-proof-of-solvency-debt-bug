# zkmerkle-proof-of-solvency-debt-bug

Proof of a debt bug in binance [zkmerkle-proof-of-solvency](https://github.com/binance/zkmerkle-proof-of-solvency) repository.

## Description of the bug

On 10.02.2022 Binance [released automated Proof of Reserves system, based on zk-SNARK](https://www.binance.com/en/blog/ecosystem/how-zksnarks-improve-binances-proof-of-reserves-system-6654580406550811626).

I checked their coud and found that it is possible to create a fake proof using this code, because there is a bug which allows setting `BasePrice` to very high value, there is missing CheckValueInRange validation for this parameter. However, BasePrice is public for everyone, so it would be easy to detect if it’s invalid or not. But there is a way to do it in a way that will be impossible to detect by other users. 

Each proof is generated for batch of 864 users, then they’re linked with each other using the following poseidon hash:
```go
// verify whether befo// verify whether beforeCexAssetsCommitment is computed correctly
for i := 0; i < len(b.BeforeCexAssets); i++ {
	CheckValueInRange(api, b.BeforeCexAssets[i].TotalEquity)
	CheckValueInRange(api, b.BeforeCexAssets[i].TotalDebt)
	cexAssets[i] = api.Add(api.Mul(b.BeforeCexAssets[i].TotalEquity, utils.Uint64MaxValueFrSquare),
		api.Mul(b.BeforeCexAssets[i].TotalDebt, utils.Uint64MaxValueFr), b.BeforeCexAssets[i].BasePrice)
	afterCexAssets[i] = b.BeforeCexAssets[i]
}
actualCexAssetsCommitment := poseidon.Poseidon(api, cexAssets...)
```
So it is basically a combination of three parameters to one huge number: `(TotalEquity << 128) + (TotalDebt << 64) + BasePrice`.

Here’s how we can abuse it, let’s say that the BasePrice of the first asset is equal to 1000. After the 1st batch we have a TotalEquity equal 0 and TotalDebt equal 1 for the first asset. 
The following value will be used to generate hash of it: `(1 << 64) + (1000)`

In the 2nd batch, instead of sending TotalDebt equal 1, we send it equal to 0, but we also change the value of BasePrice to `(1 << 64) + 1000` which will give us the same Poseidon hash. Now we can use this huge BasePrice to fake user equity/assets, a single coin with such high base price will allow to fake debt of any other coin.

In the 3td batch, we just restore TotalDebt to 1 and BasePrice to 1000, it still gives the same, correct checksum. No one is able to detect that BasePrice was changed in the 2nd block.

Now let's say exchange is listing bitcoin and has 10000 equity, and 0  debt. By using this bug, they can fake 9000 bitcoins debt, so in the end they would need to show that they hold only 1000 bitcoins instead of 10000.

Repository contains valid proof for binance proof of solvency with `BatchCreateUserOpsCounts` set to 16 (it's changed for easier testing). In the proof you can find that all original BasePrices from `sampledata` were not changed, but there is 200 mln debt of AAVE token, worth 8 bln USD. So in this proof, the debt is higher than equity, that should not be possible, which proves that there is a debt bug in the original source code.

```json
{
      "TotalEquity": 0,
      "TotalDebt": 10000000000000000,
      "BasePrice": 5640000000,
      "Symbol": "aave",
      "Index": 1
}
```

### Steps to verify proof

I generated a valid proof which is using bug, is it stored in `src/verifier/config`. To verify it, you need to generate ZKP keys (it takes few minutes), follow the instruction:

1. generate keys for ZKP
```bash
cd src/keygen && go run main.go
```
2. run verifier
```bash
cd src/verifier && go run main.go
```


### Steps to generate this proof

If you want to generate this proof by yourself then follow this steps:

1. generate keys for ZKP
```bash
cd src/keygen && go run main.go
```
2. run dockers:
```bash
docker run -d --name zkpos-redis -p 6379:6379 redis
docker run -d --name zkpos-kvrocks -p 6666:6666 apache/kvrocks
docker run -d --name zkpos-postgres -p 5432:5432 -e PGDATA=/var/lib/postgresql/data/pgdata -e POSTGRES_PASSWORD=zkpos@123 -e POSTGRES_USER=postgres -e POSTGRES_DB=zkpos postgres
```
3. run witness
```bash
cd src/witness && go run main.go
```
4. run prover
```bash
cd src/prover && go run main.go
```
5. export proof0 table for postgres zkpos database and save it to `src/verifier/config/proof.csv`
6. run verifier
```bash
cd src/verifier && go run main.go
```

### Author

b.barwikowski@hacken.io
