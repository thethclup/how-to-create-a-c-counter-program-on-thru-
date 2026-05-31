# HOW TO CREATE A C COUNTER PROGRAM ON THRU

<img width="400" height="400" alt="image" src="https://github.com/user-attachments/assets/efc20696-797b-47dc-ad26-66b8c065426b" />

## Requirements
- Buf CLI
- Rust
- Required dependencies

# SETUP ENVIRONMENT
## Install Dependencies
open up your terminal and migrate into the windows subsystem for linux (wsl) using command
<pre>
  wsl
</pre>

then 
<pre>
  sudo apt update && sudo apt install -y build-essential pkg-config libssl-dev protobuf-compiler libclang-dev clang npm
</pre>

<img width="1188" height="408" alt="install dependencies" src="https://github.com/user-attachments/assets/aeb96b1a-d7f2-4937-91b4-7f69b6c3952e" />

you should see done

## install Buf CLi
<pre>
  sudo npm install -g @bufbuild/buf
</pre>

<img width="494" height="98" alt="buf" src="https://github.com/user-attachments/assets/436a2da4-b108-488a-b6b0-c547383c473e" />

## install Rust
<pre>
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
</pre>

press 1 from the options provided and enter

<img width="1271" height="676" alt="rust" src="https://github.com/user-attachments/assets/01fecd65-5d75-4c20-9a2b-dd7e83778db8" />


run command below to set up environment
<pre>
  source $HOME/.cargo/env
</pre>

## install thru CLI
<pre>
  cargo install thru-cli --locked
</pre>

<img width="921" height="604" alt="force thru worked" src="https://github.com/user-attachments/assets/87056356-d8a7-4bbc-9343-11209a068186" />


why i used locked? (normal command encountered version error)

## make thru cli directory
1. create folder
<pre>
  mkdir -p ~/.thru/cli
</pre>

ii. generate wallet
<pre>
  thru-cli keys generate default
</pre>

iii. create and fund account (will be done automatically)
<pre>
  thru-cli account create default
</pre>

# INSTALL TOOLCHAIN
## toolchain
<pre>
  export PATH="$HOME/.cargo/bin:/usr/local/bin:/usr/bin:/bin" && thru-cli dev toolchain install --force
</pre>

<img width="1262" height="235" alt="install toolchain" src="https://github.com/user-attachments/assets/cb1c18ea-2590-4bc0-aaca-342d60d2ad6f" />

## install c sdk

<pre>
  thru-cli dev sdk install c
</pre>

<img width="1265" height="247" alt="install c sdk" src="https://github.com/user-attachments/assets/dcbc6bef-1eeb-43c7-8339-d1c75154a5d9" />


# CREATE AND COMPILE PROJECT

it is not necessary you use this same project, you can check thru documentations for any sort you want

<pre>
  # 1. Create Directories
mkdir -p ~/my-counter/examples
cd ~/my-counter

# 2. Create the Main Makefile (Using Absolute Paths)
# We capture the real Linux path to avoid /mnt/c/ errors
REAL_HOME=$(echo $HOME)
cat <<EOF > GNUmakefile
BASEDIR:=\$(CURDIR)/build
THRU_C_SDK_DIR:=$REAL_HOME/.thru/sdk/c/thru-sdk
THRU_RISCV_TOOLCHAIN:=$REAL_HOME/.thru/sdk/toolchain
include \$(THRU_C_SDK_DIR)/thru_c_program.mk
EOF

# 3. Create the Local Build Config
cat <<EOF > examples/Local.mk
\$(call make-bin,tn_counter_program_c,tn_counter_program,,-ltn_sdk)
EOF

# 4. Create the Header File (.h)
cat <<EOF > examples/tn_counter_program.h
#ifndef TN_COUNTER_PROGRAM_H
#define TN_COUNTER_PROGRAM_H
#include <thru-sdk/c/tn_sdk.h>

#define TN_COUNTER_ERR_INVALID_INSTRUCTION 0x1000UL
#define TN_COUNTER_INSTRUCTION_CREATE    (0U)
#define TN_COUNTER_INSTRUCTION_INCREMENT (1U)

typedef struct __attribute__((packed)) {
    uint instruction_type;
    ushort account_index;
    uchar counter_program_seed[TN_SEED_SIZE];
    uint proof_size;
} tn_counter_create_args_t;

typedef struct __attribute__((packed)) {
    uint instruction_type;
    ushort account_index;
} tn_counter_increment_args_t;

typedef struct __attribute__((packed)) {
    ulong counter_value;
} tn_counter_account_t;
#endif
EOF

# 5. Create the C Logic File (.c)
cat <<EOF > examples/tn_counter_program.c
#include <stddef.h>
#include <thru-sdk/c/tn_sdk.h>
#include <thru-sdk/c/tn_sdk_syscall.h>
#include "tn_counter_program.h"

static void handle_create(uchar const *data) {
    tn_counter_create_args_t const *args = (tn_counter_create_args_t const *)data;
    uchar const *proof = data + sizeof(tn_counter_create_args_t);

    if (tsys_account_create(args->account_index, args->counter_program_seed, proof, args->proof_size) != TSDK_SUCCESS) {
        tsdk_revert(TN_COUNTER_ERR_INVALID_INSTRUCTION);
    }
    tsys_set_account_data_writable(args->account_index);
    tsys_account_resize(args->account_index, sizeof(tn_counter_account_t));
    
    tn_counter_account_t* acc = (tn_counter_account_t*)tsdk_get_account_data_ptr(args->account_index);
    acc->counter_value = 0;
    tsdk_return(TSDK_SUCCESS);
}

static void handle_increment(uchar const *data) {
    tn_counter_increment_args_t const *args = (tn_counter_increment_args_t const *)data;
    tn_counter_account_t* acc = (tn_counter_account_t*)tsdk_get_account_data_ptr(args->account_index);
    
    if (acc == NULL) tsdk_revert(TN_COUNTER_ERR_INVALID_INSTRUCTION);
    
    tsys_set_account_data_writable(args->account_index);
    acc->counter_value++;
    tsys_emit_event((uchar const *)&acc->counter_value, sizeof(ulong));
    tsdk_return(TSDK_SUCCESS);
}

TSDK_ENTRYPOINT_FN void start(uchar const *data, ulong size) {
    if (size < sizeof(uint)) tsdk_revert(TN_COUNTER_ERR_INVALID_INSTRUCTION);
    uint const *type = (uint const *)data;

    if (*type == TN_COUNTER_INSTRUCTION_CREATE) {
        handle_create(data);
    } else if (*type == TN_COUNTER_INSTRUCTION_INCREMENT) {
        handle_increment(data);
    } else {
        tsdk_revert(TN_COUNTER_ERR_INVALID_INSTRUCTION);
    }
    // Required to satisfy strict compiler checks
    tsdk_revert(TN_COUNTER_ERR_INVALID_INSTRUCTION);
}
EOF
</pre>

## run it
<pre>
  make
</pre>

<img width="803" height="284" alt="make working" src="https://github.com/user-attachments/assets/cf99935b-9667-4fa2-80ef-84b5b4a08277" />

# DEPLOY AND INTERACT WITH YOUR PROGRAM
## create account
<PRE>
  thru-cli program create my_counter_v1 ./build/thruvm/bin/tn_counter_program_c.bin
</PRE>

<img width="1003" height="455" alt="create program account" src="https://github.com/user-attachments/assets/5e067e31-757a-4854-8c98-7f8f47334c8a" />

copy everything from the results this line provides and paste in your notepad, you will make use of figures attached to the program account [program id]

## create counter address
<pre>
  thru-cli program derive-address <PROGRAM_ID> count_acc
</pre>

<img width="947" height="72" alt="generate derived address" src="https://github.com/user-attachments/assets/20ae25e6-8e87-4a3d-bab0-d5e20fef43ca" />

input your program account as the PROGRAM ID
after running the command you should see a derived address, copy it too. this derived address will serve as counter ID

## create the python script and run it
i. create the script
<pre>
  nano deploy.py
</pre>

a new environment opens up, the code below by right clicking
<pre>
  import subprocess
import sys

# REPLACE THESE WITH YOUR ADDRESSES
PROGRAM_ID = "PUT_YOUR_PROGRAM_ID_HERE"
COUNTER_ID = "PUT_YOUR_COUNTER_ID_HERE"

def run_command(cmd):
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    return result.stdout.strip()

print(f"1. Generating Proof for {COUNTER_ID}...")
output = run_command(f"thru-cli txn make-state-proof creating {COUNTER_ID}")

proof_hex = ""
for line in output.split('\n'):
    if "Proof Data (hex):" in line:
        proof_hex = line.split(":")[1].strip()
        break

if not proof_hex:
    print("ERROR: Could not find Proof Data.")
    sys.exit(1)

# Construct Payload: Prefix + Size + Proof
prefix = "000000000200636F756E745F6163630000000000000000000000000000000000000000000000"
size_hex = int(len(proof_hex) // 2).to_bytes(4, 'little').hex()
full_payload = prefix + size_hex + proof_hex

print("2. Executing Transaction...")
cmd = f"thru-cli txn execute --fee 0 --readwrite-accounts {COUNTER_ID} {PROGRAM_ID} {full_payload}"
print(run_command(cmd))
</pre>

remember to input your program ID and counter ID in the code block

CTRL O, THEN "ENTER" AND FINALLY CTRL X to exit

## run the program
<pre>
  python3 deploy.py
</pre>

## run increment also
<pre>
  thru-cli txn execute \
  --fee 0 \
  --readwrite-accounts <COUNTER_ID> \
  <PROGRAM_ID> \
  010000000200
</pre>

<img width="852" height="354" alt="increment counter" src="https://github.com/user-attachments/assets/252fcabf-c26c-46c5-9737-7960339be2b7" />

# test if your program worked

visit: scan.thru.org
then paste either your counter id or program id

![Screenshot_22-1-2026_17527_scan thru org](https://github.com/user-attachments/assets/f1ecffd6-4aeb-4085-90ca-aec384e80a6c)


# THAT WILL BE ALL

# thanks for using my guide

any issues can be forwarded to me on X dms 


