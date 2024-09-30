# Preparing Phase 2 Data Files

These notes cover the procedure for converting phase 1 files (i.e. the data files we normally generate with PPoT contributions) into data that is ready for use in a phase-2 circuit-specific trusted setup.

This process does not add secret entropy to the phase 1 files. It is deterministic in nature, and be re-run if problems occur during the process. 

The process is, in overview:
- take a phase 1 file
- apply a beacon
- run a 'prepare phase 2' operation
- truncate the result, which has the full 2^28 powers of tau, creating a file for each lower power.

The process is lengthy, so is not done too often. It would be nice to make a routine of doing this every 5 contributions, or some such schedule. No doubt this will depend on the priorities of the team members involved.

## Machine Setup
The process requires a great deal of computation and of disk storage. Choose a machine with appropriate capacity. I suggest a 64-core compute-optimised EC2 machine such as a c6i.16xlarge. The full 2^28 prepared file is 382gb. The phase 1 files that will be used for preparation are round 100gb, so it's necessary to have around 500gb storage as a minimum.

Its recommended to use the EC2 image template `ppot-prepare`. 

The machine will need:
- AWS CLI utilities
- Node.js
- snarkjs

Most of the commands below are long-running. Run them in a `screen` session so that you can return to the command if your connection breaks. 

Note the beacon announcement step mentioned below. You might want to take care of that before starting up the machine. 

## Convert to .ptau
The preparation steps use files in the `snarkjs ptau` format. Phase 1 contribution data files are stored in the format used by https://github.com/kobigurk/phase2-bn254 (Rust code). The Rust version has commands for preparing and truncating phase 2 files, but the resulting files don't have the public key history embedded, as the `ptau` format does. Nor is there an export/import command to convert the file to `ptau` format. Snarkjs and its `ptau` files are widely used by client projects, so we get both a popular format and a verifiable history by preparing `ptau` phase 2 files.

Snarkjs does have import commands for phase 1 files in the Rust format. We use that first to converty our chosen contribution file to `ptau` format.

Example:
```
$ snarkjs powersoftau import response pot28_0098_nopoints.ptau response_0099 pot28_0099_foobar.ptau -v --name=foobar
```
This command performs these steps:
- Loads the public key history prior to contribution #99.
- Loads the powers-of-tau points from the `response` file for contribution #99, along with its public key.
- Tags the new public key with the value of the `--name` argument. 
- Outputs a `ptau` file with both the points and the public key history. 

## The Beacon
Experts differ as to whether the beacon step adds any security, but it is required by `snarkjs`, so we do include this step. 

The beacon should be derived from a source of randomness such as the Ethereum beacon chain. Choose a point in time at least a day or two in the future, and calculate a slot number (usually a round number) around that time from which the randomness will be taken. Announce that slot number on some public forum, something like: 'The beacon hash for PPoT phase 2 files prepared from contribution #xx will be derived from the RANDAO reveal of the Ethereum beacon chain at slot xxxxxxxxx.' 

Wait for that slot to be finalised, then look up the `RANDAO reveal` value. Use that value (hex digits only) as the beacon hash value in the `snarkjs powersoftau beacon` command. Example: 
```
node --max-old-space-size=204800 /usr/bin/snarkjs powersoftau beacon ppot_0080.ptau ppot_0080_beacon.ptau af941599f6d640b5b4b6116d3ded861b3362a964c390edc270aef45ba17b67148fb3d7ab901a68b1528c9bb3e16721cc000dda5d8466f4aa4a1c8ca9eb57d05e6c2d2e780d6a793df90a1ebd076bb3dd9b7d4075e3e68b36b86c1fb7c4feeded 31 -v
```

## Prepare Phase 2

The `snarkjs powersoftau prepare phase2` command is used to take the file with the beacon contribution and generate a full phase-2-ready file.

When dealing with the number of points we have, this command has several pitfalls. 

### *Memory* 

Use the Node.js options, mainly `--max-old-space-size` to prevent the process running out of memory.

Create a swap file. Each section is processed in memory. You won't need the full 380gb to reside in memory at once, but you will need a sizable portion of that. 

See also [this](https://github.com/hermeznetwork/phase2ceremony_4/blob/main/VERIFY.md), which has another approach to memory issues.

See also [this](https://hackmd.io/SUlyJfrNTDqBkSyGJ7E0cQ)


### *Too many elements passed to Promise.all*

The number of points we process exceeds a limit inherent in the underlying libraries on which `Promise.all` is based. 
A fix is to install this branch of `fastfile`: 
https://github.com/zkparty/fastfile/tree/limit-promises
... which structures an array of Promise arrays so as to avoid the limit.

------------
Clone the snarkjs branch and the fastfile branch to folders on the disk where you'll porocess the data. Build both of them.

Use the `npm link` command in the fastfile directory to make it the global module. Use `npm link fastfile` in the snarkjs folder to link to it instead of the downloaded package. Rebuild snarkjs.

---------

### *Split and Merge*

To avoid a cycle of long-running attempts that may end in failure, we have a fork of snarkjs that allows the process to be broken up into shorter-running chunks. 

The fork is [here](https://github.com/glamperd/snarkjs/tree/run-prepare). Use the `run-prepare` branch. 

This approach breaks the job into segments, corresponding to the sections in the ptau file, namely:

| Section | Section No. | Prepared Section No. |
|---------|--------|------------|
| Tau.g1 | 2 | 12 |
| Tau.g2 | 3 | 13 |
| AlphaTau.g1 | 4 | 14 |
| BetaTau.g1 | 5 | 15 |

We can further divide the job within each section into each level of powers of 2. Each section has 2^28 powers except that Tau.g1 has almost 2^29 and BetaTau.g2 has only 1 point.

Each increment in the exponent has twice as many points. So, for example, the 2^27 exponent has as many points as 1 to 26. The command line has arguments to select the exponent range to process. The job can be chunked into chunks of a size of your choice.

After processing, the generated data can be merged in to a final prepared file. This must be done in order sequence with respect to the exponents. Snarkjs will only allow merging into a single final file, i.e. the chunks can't be merged together, then merged again to the final file.

Example commands:
```
export NODE_OPTIONS=--max-old-space-size=480800
./build/cli.cjs powersoftau prepare section ppot_0080_beacon.ptau ppot_0080 4 0 28 -v
```
The above command processes section 4 (AlphaTau.g1), and powers 0 to 28, which in this case is the whole section. The generated file will be named starting with `ppot_0080`. The newly generated section (14 in this case) and exponent range (0 to 28) will be automatically appended.

```
 snarkjs]$ ./build/cli.cjs powersoftau prepare merge ppot_0080_prepared.ptau ppot_0080_s13_27_27.ptau ppot_0080_s13a.ptau -v
```
The above command will merge a prepared chunk (ppot_0080_s13_27_27.ptau) into a partial final file (ppot_0080_s13_27_27.ptau) to create a new partial final file (ppot_0080_s13a.ptau).

The end result once all chunks have been processed and merged is a full prepared file, just as if the `powersoftau prepare phase2` command had been run. The intermediate files can be discarded. 

### Wrapping up

The final step is to truncate the prepared file. Use the `snarkjs powersoftau truncate` command. You'll then have a full set of files to publish, 1 for each exponent up to 28.




