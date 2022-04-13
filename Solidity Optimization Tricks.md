# Solidity Gas Optimization Tips

---

##### Tip 1

**`for`** loop Optimize

```solidity
for(uint i=0; i<arr.length; i++){       //Bad All Interation Calculate the arr.length
    // do something
}
```

```solidity
uint memory length = arr.length // Calculate arr.length once
for(uint i=0; i<length;i= unchecked_inc(i) {       //Good
    // do something
}

function unchecked_inc(uint i) internal returns(uint) {
    unchecked {
        return i;
    };
}
```

Note that it’s important that the call to unchecked_inc is inlined. This is only possible for solidity versions starting from `0.8.2`.

##### Tip 2

`Array` Optimize

1. Use the Fixed Size array instead of Dynamaic Array
2. If Possible Use the Mapping instead of Array

##### Tip 3

`Operation` Optimize

```solidity
uint a = 2 ;
uint b = a / 2;     // Bad
uint c = a / 4;
uint d = a * 8;
```

```solidity
uint a = 2 ;
uint b = a >> 1;     // Good
uint c = a >> 2;
uint d = a << 3;
```

While the `DIV / MUL` opcode uses 5 gas, the `SHR / SHL` opcode only uses 3 gas. Furthermore, beware that Solidity's division operation also includes a division-by-0 prevention which is bypassed using shifting. Eventually, `overflow checks are never performed for shift operations` as they are done for arithmetic operations. Instead, the result is always truncated.

##### Tip 4

`Calldata` vs `Memory` Optimize

```
 function add(uint[] memory arr) external returns (uint sum) {
        uint length = arr.length;
        for (uint i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
    }
```

In the above example, the dynamic array arr has the storage location memory . When the function gets called externally, the array values are kept in calldata and copied to memory during ABI decoding (using the opcode `calldataload` and `mstore` ). And during the for loop, arr[i] accesses the value in memory using a `mload.`

```
function add(uint[] calldata arr) external returns (uint sum) {
        uint length = arr.length;
        for (uint i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
    }
```

In the former example, the ABI decoding begins with copying value from `calldata` to `memory` in a for loop. Each iteration would cost at least `60 gas`. In the latter example, this can be completely avoided. This will also reduce the number of instructions and therefore reduce the deployment time cost of the contract.

> Note : use `calldata` instead of `memory` if the function argument is only read.

##### Tip 5

`variable` declaration Order

Solidity contracts have contiguous 32 byte (256 bit) slots used for storage. When we arrange variables so multiple fit in a single slot, it is called variable packing.

```
uint128 a;
uint256 b;    //bad Assign Total 3 slot
uint128 b;
```

Variable packing is like a game of Tetris. If a variable we are trying to pack exceeds the 32 byte limit of the current slot, it gets stored in a new one. We must figure out which variables fit together the best to minimize wasted space.

```
uint128 a;
uint128 c; //Good Assign only 2 slot
uint256 b;
```

These variables are packed. Because packing `c` with `a` does not exceed the 32 byte limit, they are stored in the same slot.

This small change will save you a lot of gas as it will now only need 2 slots to store (It takes 20k gas to store 1 slot of data).

##### Tip 6

`require` Optimize

```
require(a > 0 , "a is less than  or Equals to Zero.")         //Good

require(a>0 , "a is less than  or Equals to Zero. Please enter the value greater
than Zero.")                        // Bad

```

You can (and should) attach error reason strings along with require statements to make it easier to understand why a contract call reverted. These strings, however, take space in the deployed bytecode. Every reason string takes at least 32 bytes so make sure your string fits in 32 bytes or it will become more expensive.
