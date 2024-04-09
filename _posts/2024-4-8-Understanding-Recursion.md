---
layout: post
title: How recursing works?
date: 2024-04-08 14:50
summary: This essay explores recursion and tail recursion optimization in depth. It compares non-tail and tail recursive functions using assembly code examples, demonstrating the potential for optimization in the latter. The paper concludes by discussing the benefits and limitations of manual and compiler-based tail recursion optimization, providing a comprehensive understanding of recursion in programming.
categories: recursing algorithm stack
---

## Understanding Recursion

Let's consider a typical example of recursion:

```c
int factorial(int n) {
	if (n == 0) {
		return 1;
	} else {
		return n * factorial(n - 1);
	}
}
```

Here, the function calls itself, which is a characteristic of recursive methods. A more precise definition of recursion can be found on [Wikipedia](https://en.wikipedia.org/wiki/Recursion_(computer_science)).

## Exploring the Stack

Non-tail recursion often cannot be optimized, so let's examine the corresponding assembly code.

```asm
    1138:       f3 0f 1e fa             endbr64
    113c:       55                      push   %rbp
    113d:       48 89 e5                mov    %rsp,%rbp
    1140:       48 83 ec 10             sub    $0x10,%rsp
    1144:       89 7d fc                mov    %edi,-0x4(%rbp)
    1147:       83 7d fc 00             cmpl   $0x0,-0x4(%rbp)
    114b:       75 07                   jne    1154 <factorial+0x1c>
    114d:       b8 01 00 00 00          mov    $0x1,%eax
    1152:       eb 11                   jmp    1165 <factorial+0x2d>
    1154:       8b 45 fc                mov    -0x4(%rbp),%eax
    1157:       83 e8 01                sub    $0x1,%eax
    115a:       89 c7                   mov    %eax,%edi
    115c:       e8 d7 ff ff ff          call   1138 <factorial>
    1161:       0f af 45 fc             imul   -0x4(%rbp),%eax
    1165:       c9                      leave
    1166:       c3                      ret
```

The prologue of this function sets up the stack frame:

```asm
	1138:       f3 0f 1e fa             endbr64
	113c:       55                      push   %rbp
	113d:       48 89 e5                mov    %rsp,%rbp
	1140:       48 83 ec 10             sub    $0x10,%rsp
	1144:       89 7d fc                mov    %edi,-0x4(%rbp)
```

The epilogue cleans up the stack frame and returns to the caller:

```asm
	1165:       c9                      leave
	1166:       c3                      ret
```

By visualizing the stack, we can see the stack frame of the function `factorial` as follows:

```
	+----------------+
	| return address | 
	+----------------+
	| old %rbp       | 
	+----------------+
	| n              | 
	+----------------+
```

After the second call of `factorial`, the stack frame is extended:

```
	+----------------+
	| return address | 
	+----------------+
	| old %rbp       | 
	+----------------+	                            
	| n              | 
	+----------------+						   	    
	| return address | 
	+----------------+							    					
	| old %rbp       | 
	+----------------+
	| n-1            | 
	+----------------+
```

When `n` decreases to 0, the stack frame looks like this:

```
	+----------------+
	| return address | 
	+----------------+
	| old %rbp       | 
	+----------------+	                            
	| n              | 
	+----------------+						   	    
	| return address | 
	+----------------+							    					
	| old %rbp       | 
	+----------------+
	| n-1            | 
	+----------------+
	...
	...
	...
	+----------------+
	| return address | 
	+----------------+							    					
	| old %rbp       | 
	+----------------+
	| 1              | 
	+----------------+
	| return address |
	+----------------+
	| old %rbp       |
	+----------------+
	| 0              |
	+----------------+
```

When `n` reaches 0, `eax` is set to 1, and the function returns, popping the top frame. The remaining stack frames return to this line:

```asm
	1161:       0f af 45 fc             imul   -0x4(%rbp),%eax
```

This line multiplies the `%eax` with the return value of the next frame.

In essence, the procedure is as follows: the function creates stack frames for values from `n` to 0. Then, it returns from 0 to `n`, multiplying the return value with the `n` in each frame.

## Tail Recursion Optimization

An alternative version of the previous function that employs tail recursion is as follows:

```c
int factorial(int m, int n)
{
    if (m <= 1)
        return n;
 
    return factorial(m - 1, m * n);
}
```

The corresponding assembly code is:

```asm
000000000000114e <factorial>:
    114e:       f3 0f 1e fa             endbr64
    1152:       55                      push   %rbp
    1153:       48 89 e5                mov    %rsp,%rbp
    1156:       48 83 ec 10             sub    $0x10,%rsp
    115a:       89 7d fc                mov    %edi,-0x4(%rbp)
    115d:       89 75 f8                mov    %esi,-0x8(%rbp)
    1160:       83 7d fc 01             cmpl   $0x1,-0x4(%rbp)
    1164:       7f 05                   jg     116b <factorial+0x1d>
    1166:       8b 45 f8                mov    -0x8(%rbp),%eax
    1169:       eb 16                   jmp    1181 <factorial+0x33>
    116b:       8b 45 fc                mov    -0x4(%rbp),%eax
    116e:       0f af 45 f8             imul   -0x8(%rbp),%eax
    1172:       8b 55 fc                mov    -0x4(%rbp),%edx
    1175:       83 ea 01                sub    $0x1,%edx
    1178:       89 c6                   mov    %eax,%esi
    117a:       89 d7                   mov    %edx,%edi
    117c:       e8 cd ff ff ff          call   114e <factorial>
    1181:       c9                      leave
    1182:       c3                      ret
```

The initial stack frame is set up as before:

```
	+----------------+
	| return address | 
	+----------------+
	| old %rbp       | 
	+----------------+	                            
	| n              | 
	+----------------+						   	    
	| m              | 
	+----------------+

```
> Note that the arguments are passed in reverse order.

The final stack frame appears as follows:

```
	+----------------+
	| return address | 
	+----------------+
	| old %rbp       | 
	+----------------+	                            
	| n              | 
	+----------------+						   	    
	| m              | 
	+----------------+
	| return address | 
	+----------------+							    					
	| old %rbp       | 
	+----------------+
	| m * n          | 
	+----------------+
	| m - 1          |
	+----------------+
	...
	...
	...
	+----------------+
	| return address |
	+----------------+
	| old %rbp       |
	+----------------+
	| m!             |
	+----------------+
	| 1			     |
	+----------------+
```
Unlike non-tail recursion, the local variable of a previous stack frame is not involved in the computation of the current stack frame. Therefore, an optimization is possible by reusing the same stack frame for all recursive calls. We can simply discard the previous stored values and update the current stack frame with the new values.

The optimized version of the assembly code, achieved by using the `-O3` flag in gcc, is as follows:

```bash
gcc -O3  tail_recur_factl.c
objdump -d a.out
```

Optimized version:

```asm
0000000000001140 <factorial>:
    1140:       f3 0f 1e fa             endbr64
    1144:       89 f0                   mov    %esi,%eax
    1146:       83 ff 01                cmp    $0x1,%edi
    1149:       7e 10                   jle    115b <factorial+0x1b>
    114b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)
    1150:       0f af c7                imul   %edi,%eax
    1153:       83 ef 01                sub    $0x1,%edi
    1156:       83 ff 01                cmp    $0x1,%edi
    1159:       75 f5                   jne    1150 <factorial+0x10>
    115b:       c3                      ret
```

As evident, the optimized version is significantly simpler and more efficient than the non-tail recursive version.

Interestingly, when we apply the `-O3` optimization to the non-tail recursive version, we find that it too is optimized to a tail recursive version.

```bash
0000000000001140 <factorial>:
    1140:       f3 0f 1e fa             endbr64
    1144:       b8 01 00 00 00          mov    $0x1,%eax
    1149:       85 ff                   test   %edi,%edi
    114b:       74 0b                   je     1158 <factorial+0x18>
    114d:       0f 1f 00                nopl   (%rax)
    1150:       0f af c7                imul   %edi,%eax
    1153:       83 ef 01                sub    $0x1,%edi
    1156:       75 f8                   jne    1150 <factorial+0x10>
    1158:       c3                      ret
```

Is it unnecessary to manually convert non-tail recursive functions into tail recursive ones? The answer is nuanced. Modern compilers possess the capability to optimize non-tail recursive functions, but this doesn't render manual tail recursion obsolete. There are several reasons for this.

+ Firstly, tail recursion can enhance the readability of your code, making it more comprehensible. This is particularly true when dealing with complex algorithms where understanding the flow of execution is critical.

+ Secondly, there can be performance benefits from tail recursion in situations where the compiler's automatic optimization is not applicable or insufficient. Tail recursion uses constant stack space and can therefore be more efficient than non-tail recursion, which can use stack space proportional to the depth of recursion.

+ Lastly, there are cases where the compiler may not be able to automatically convert a non-tail recursive function into a tail recursive one. This is especially true for functions where the recursion is more complex, such as when multiple recursive calls are made, or when the recursive call is not the final operation in the function. In such scenarios, manual conversion to tail recursion can be necessary for optimizing performance.