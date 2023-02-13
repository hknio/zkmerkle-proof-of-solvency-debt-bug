# zkmerkle-proof-of-solvency-debt-bug

Proof of concepct of debt bug in binance [zkmerkle-proof-of-solvency](https://github.com/binance/zkmerkle-proof-of-solvency) source code.

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

### Steps to generate this proof

1. generate keys for ZKP
```bash
cd src/keygen && go run main.py
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
5. export proof0 table for postgres zkpos database and save it to `src/verifier/proof.csv`
6. run verifier
```bash
cd src/prover && go run main.go
```

You can skip steps 2-5 if you do not want to generate proof by yourself, there is already my proof in `src/verifier`.

### Author

b.barwikowski@hacken.io
