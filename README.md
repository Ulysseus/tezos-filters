I run a small baking service https://youloafwebake.io, I hope some of the following filters may be useful to anyone baking

jq is a utility for reading JSON output https://stedolan.github.io/jq/
Tezos offers some RPC commands that return a JSON object of information from the running node.
jq allows one to filter the results into a more usable form, this can then be used in a bash script to help with monitoring of the node.
I use Mathematica for most of my work, but my node runs in Ubuntu 18.04, the rest of the time I use a Mac.
I hope these filters are useful to people

First a few things > and >> in Ubuntu mean redirect output; > means overwrite >> means append to end of file.
So the command <do something> > <filename> will redirect the output of <do something> to <filename>. >> would append the output to <filename>.
The following link has all the RPCs currently available
https://tezos.gitlab.io/betanet/api/rpc.html

The following RPC ./tezos-client rpc get /network/peers?filter='running' >> test.json
appends a list of running peers to test.json
The command `jq '[.[][0]] | length' test.json` will return the number of running peers, I use this command to determine if the running peer count ever drops to zero.
By running the RPC every few minutes on the node I can record a log of the active peer count.

To do this I use a bash script like follows

```
#!/bin/sh
while true
do
$HOME/tezos/./tezos-client rpc get /network/peers?filter='running' >> test.json
sleep 3000
done
```
Saving this as file as say check_peers and then adding executable status for all users by typing 
`sudo chmod a+x check_peers`
The script is then run by typing 
`./check_peers`
in a new terminal window.


The following RPC will return the number of unique bakers in cycle 21:
```
./tezos-client rpc get /chains/main/blocks/head/helpers/baking_rights?'cycle=21&max_priority=1'| jq '[.[].delegate] | unique | length'
```

The RPC: 
```
./tezos-client rpc get /chains/main/blocks/head/context/delegates/tz1eZwq8b5cvE2bPKokatLkVMzkxz24z3Don > test.json
```
gives you all the information about a baking address

Then the following jq filters will extract the information
1. `jq '[.frozen_balance_by_cycle[].rewards | tonumber] | add' test.json` : Total the rewards per cycles of the contract
2. `jq '.delegated_contracts[]' test.json` : All contracts that are delegating to the Baker
3. `jq '[.frozen_balance_by_cycle[].deposit | tonumber] | add' test.json` : Total of deposits
4. `jq '[.frozen_balance_by_cycle[].fees | tonumber] | add' test.json` : Total of fees earned
5. `jq '.balance' test.json` : Get Balance of the Baker
6. `jq '.frozen_balance' test.json` : Get Frozen balance of the Baker
7. `jq '[.delegated_balance]' test.json` : The delegated balance of the Baker
8. `jq '[.delegated_contracts[]] | unique | length' test.json` : Number of unique addresses delegating

The previous RPC gets the information about the head block the following RPC gets the information from a block an integer number (in this case 60 000 levels) before the head.
```
./tezos-client rpc get /chains/main/blocks/head\~60000/context/delegates/tz1eZwq8b5cvE2bPKokatLkVMzkxz24z3Don > test.json
```
You can also put a block-id like `BM4TJHwLhjNz2UkiMwAb2He9gr7LcxG5bkNsPqyw4zGd18sc6wy\~60000` instead of head.
Notice its a tilde sign `\~` not a minus sign `-`

The following bash script automates the procedure
This bash script automates the procedure (need to do sudo chmod a+x 'filename')

```
#!/bin/sh 
i=0
while true
do
i=$((i+10000))
sleep 10
$HOME/tezos/./tezos-client rpc get /chains/main/blocks/head~$i/context/delegates/tz1eZwq8b5cvE2bPKokatLkVMzkxz24z3Don >> test.js
done
```

I use a script like this to jump back a cycle at a time to calculate what delegates I have and therefore what rewards are due to each.
I have not put the script up, I am not absolutely certain its right :-), but you get the idea.

## Network filters
`./tezos-client rpc get /network/connections > test.json`
1. `jq '.[].peer_id' test.json` : Gives list of connected peers
2. `jq '.[].id_point' test.json` : Gives list in same order as before of IP and port Number

## More on Baking Rights
```
./tezos-client rpc get /chains/main/blocks/head/helpers/baking_rights?"cycle=$myCycle&delegate=tz1eZwq8b5cvE2bPKokatLkVMzkxz24z3Don&max_priority=2" > node_baking_rights.json
```

Previous command will get the data from the node about baking
Now lets filter it to a csv file

# Write Headers
`jq  -r '.[-1]| keys_unsorted|@csv'  node_baking_rights.json > node_baking_rights.csv`

#Append data
```
jq -r  '.|map([.level,.delegate,.priority,.estimated_time])|.[] | @csv' node_baking_rights.json >> node_baking_rights.csv
```

## Now for Past Data
```
./tezos-client rpc get /chains/main/blocks/head/context/delegates/tz1eZwq8b5cvE2bPKokatLkVMzkxz24z3Don > node_past_rewards.json
```
# Write headers
```
jq  -r '.frozen_balance_by_cycle[0]| keys|@csv' node_past_rewards.json > node_past_rewards.csv
```
# Append Data
```
jq -r  '[.frozen_balance_by_cycle[]]|map([(.cycle),(.deposit|tonumber),(.fees|tonumber),(.rewards|tonumber)])|.[] | @csv' node_past_rewards.json >> node_past_rewards.csv 
```

## Endorsing rights
```./tezos-client rpc get /chains/main/blocks/head/helpers/endorsing_rights?"cycle=$myCycle&delegate=tz1eZwq8b5cvE2bPKokatLkVMzkxz24z3Don" > node_endorsing_rights.json
```

# Write Headings
```
jq -r '[.[-1]|keys_unsorted[]]|[.[0],.[1],.[3],.[2]]|@csv' node_endorsing_rights.json > node_endorsing_rights.csv
```

# Append Data
```
jq -r  '.|map([.level,.delegate,.estimated_time,(.slots|.[])])|.[] |@csv' node_endorsing_rights.json >> node_endorsing_rights.csv
```

## Some filters for tzscan

# Read Data
```
curl --fail --silent --show-error https://api6.tzscan.io/v2/rewards_split/tz1eZwq8b5cvE2bPKokatLkVMzkxz24z3Don?cycle=$myCycle > tzscan_rewards_split.json
```
# Write Headings
```
echo "Contract", "Balance", "Cycle", $myCycle > tzscan_contract_balances.csv
```
# Append Data
```
jq -r '[.delegators_balance[]]|map([.[0].tz,(.[1]|tonumber)])|.[]|@csv' tzscan_rewards_split.json >> tzscan_contract_balances.csv
```
