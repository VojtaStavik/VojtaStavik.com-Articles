---
layout: post
title: Hamming Distance - Programming Interview Problem (I.)
filename: "2017-03-02-swift-programming-interview-hamming-distance.md"
---

This is the first post of a small series focused on common technical interview challenges. You can find a lot of articles and StackOverflow answers on the similar topic. However, I would like to focus on a Swift-first approach and detailed explanation. Fore the sake of curiosity, I will also show an Objective-C version of the final algorithm. I'm going to use problems from the awesome website [LeetCode](https://leetcode.com/problemset/algorithms/). So let's start!

![Hamming distance - 3 bits](https://upload.wikimedia.org/wikipedia/commons/6/6e/Hamming_distance_3_bit_binary_example.svg)
*(Image source: Wikipedia)*

**The problem:** Given two integers ```x``` and ```y```, calculate the Hamming distance. The Hamming distance between two integers is the number of positions at which the corresponding bits are different.

<!-- more -->

Every integer can be represented in a binary form as a sequence of ```0``` and ```1```. To find out more about how the binary representation of a given number works, go to this [Wikipedia page](https://en.wikipedia.org/wiki/Binary_number).

Let's say we have two integers, 2 and 6. Their binary representations look like this:
{% highlight swift %}
  2 ==>  0 1 0
  6 ==>  1 1 0
{% endhighlight %}

We need to compare the numbers and count, how many times are the bits on the corresponding positions different:
{% highlight swift %}
      2 ==>  0 1 0
      6 ==>  1 1 0
  ----------------
  Different? ✓ ✗ ✗
{% endhighlight %}

You can see that there's only one bit which is different. **The hamming distance of numbers 2 and 6 is therefore 1.**

### Solution

Let's try to convert what we did above to a program. We have a function ```calculateHammingDistance``` and we need to write its implementation.

We can split the problem into two sub-problems:

{% highlight swift %}
  func calculateHammingDistance(x: Int, y: Int) -> Int {
    // Step 1: Find different bits

    // Step 2: Count them

  }
{% endhighlight %}


#### Step 1: Find different bits

The most obvious solution for this would be to iterate through bits of both numbers and compare them bit by bit. Fortunately, we don't have to do that. There's an operator in Swift which does exactly that!

Ladies and gentlemen, let me introduce you Bitwise XOR or ```^``` if you will:

![Bitwise XOR](/images/2017-03-02/bitwiseXOR.png)
*(Image source: Apple.com)*

Thanks to this operator, finding different bits of two numbers is just one line of code:
{% highlight swift %}
  func calculateHammingDistance(x: Int, y: Int) -> Int {
    // Step 1: Find different bits
    let differentBits = x ^ y

    // Step 2: Count them

  }
{% endhighlight %}




#### Step 2: Count the number of different bits
Let's take a look on how the ```differentBits``` variable looks like with our example numbers 2 and 6:

{% highlight swift %}
      2 ==>  0 1 0
      6 ==>  1 1 0
  ----------------
  2 ^ 6 ==>  1 0 0 // 100 in binary represents number 4
{% endhighlight %}

Before we proceed, we need to know about a few other interesting operators for bit manipulations in Swift:
{% highlight swift %}
  x >> a
  // Bitwise shift: shifts all bits to the right of 'a' positions
  // (operator << to the left)
  Example:
         8 ==>  1 0 0 0
    8 >> 1 ==>  0 1 0 0
    8 >> 2 ==>  0 0 1 0
    8 >> 3 ==>  0 0 0 1
    8 >> 4 ==>  0 0 0 0
{% endhighlight %}
![Bitwise shift](/images/2017-03-02/bitshiftUnsigned.png)
*(Image source: Apple.com)*

**Important:** The right shifting behavior is different for negative numbers. If the number is greater than zero, the most left bit will be filled with 0 after the shift. For negative numbers, the most left bit will be filled with 1.
![Bitwise shift](/images/2017-03-02/bitshiftSigned.png)
*(Image source: Apple.com)*

{% highlight swift %}
  x & y
  // Bitwise AND: Combines the bits of two numbers.
  // It returns a new number whose bits are set to 1 only
  // if the bits were equal to 1 in both input numbers

  Example:
       7 ==>  0 1 1 1
       3 ==>  0 0 1 1
       --------------
   7 & 3 ==>  0 0 1 1
{% endhighlight %}
![Bitwise AND](/images/2017-03-02/bitwiseAND.png)
*(Image source: Apple.com)*

Our strategy for counting the number of ```1``` in ```differentBits``` will be to shift its bits step by step to the right, and check if the last bit is ```1```. If we find ```1```, we increment a counter, otherwise we continue until there is no ```1``` in ```differentBits```. In the other words, until ```differentBits == 0```.

{% highlight swift %}
func calculateHammingDistance(x: Int, y: Int) -> Int {

  // Step 1: Find different bits
  let signedDifferentBits = x ^ y

  // We bitcast Int to UInt to allow the algorithm work correctly also for negative numbers.
  var differentBits: UInt = unsafeBitCast(signedDifferentBits, to: UInt.self)

  // Step 2: Count them
  var counter = 0

  // Until there are some bits set to '1' in differentBits.
  while differentBits > 0 {

      // Mask differentBits with number 1 aka 00000001.
      // By doing this, we set all bits of differentBits
      // but the last one to zero.
      let maskedBits = differentBits & 1

      // If the last bit is not zero,
      if maskedBits != 0 {
          // increment the counter.
          counter += 1
      }

      // Shift bits in differentBits to the right.
      differentBits = differentBits >> 1
  }

  // We're done, return the number of 1s we've found.
  return counter
}
{% endhighlight %}


#### Objective-C version
{% highlight swift %}
+ (NSInteger) calculateHammingDistanceFor:(NSInteger)x and:(NSInteger)y {
    // Step 1: Find different bits
    NSUInteger differentBits = x ^ y;

    // Step 2: Count them
    NSUInteger counter = 0;
    while (differentBits > 0) {
        NSUInteger maskedBits = differentBits & 1;
        if (maskedBits != 0) {
            counter++;
        }
        differentBits = differentBits >> 1;
    }

    return counter;
}
{% endhighlight %}

#### Summary

There are no specific edge cases we need to test our algorithm for. We don't need to know if the integer is negative, IntMax or zero. We're working with the integers on the bit level, and we don't really care about the numbers they represent. There's no possibility of overflow, because we're only comparing already existing integers.

**Trick question:** What is the maximal value of hamming distance for two 32bit integers?

Next time, we're going to talk about [Array rotations](/2017/03/26/swift-programming-interview-array-rotation/). Stay tuned.
