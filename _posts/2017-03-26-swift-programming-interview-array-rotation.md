---
layout: post
title: Array Rotation - Programming Interview Problem (II.)
filename: "2017-03-26-swift-programming-interview-array-rotation.md"
---

Today, we're going to focus on another famous programming interview problem - Array rotation. We start with a simple "obvious" solution and move to more optimized and interesting one afterwards.

![Why did programmer quit his job?](/images/2017-03-26/why-did-programmer-quit.jpg)

**The problem:** Rotate an array of n elements to the right by k steps. For example, with ```n = 6``` and ```k = 3```, the array ```[1,2,3,4,5,6]``` is rotated to ```[4,5,6,1,2,3]```.

<!-- more -->

### Edge cases
Let's start by thinking about possible edge cases that can happen:

```n == 1``` - The array has only one element. Any number of the array rotations will always result to the identical array.

```n == 0``` - Similar situation as above. When the array is empty, rotations don't affect it.

```k == 0``` - Rotate an array by "0" steps to the right does nothing to the array. The resulting array is identical to the input array.

```k < 0``` - What if k is negative? To rotate an array by -1 steps to the right means to rotate it by 1 step to the left. Let's say we have an array with 3 elements ```[1,2,3]```. When we rotate it by one step to the left we get ```[2,3,1]```. You can notice that we get the same result if we rotate the array by 2 steps to the right. We can easily transform left rotation to right rotation using ```stepsToRight = arrayLength - stepsToLeft```. Because we know k is already negative we should rather use ```stepsToRight = arrayLength + stepsToLeft```.

```abs(k) > n``` - Imagine we should rotate an array of 2 elements by 3 steps to the right. After two steps, the array will be identical to the original array. It means that when the array is rotated by 3 steps, it's effectively rotated just by 1 step. What if we were to rotate the array by 5 steps? After 2 rotations, we would get the identical array and 3 rotations to go. Again, we would rotate the array effectively just by 1 step. It means that:
{% highlight swift %}
 effectiveRotations = rotations % numberOfElementsInTheArray
                  1 = 5         % 2

 // We can generalize this into:
 effectiveRotations = k % n {% endhighlight %}

 We transform these edge cases to code:
{% highlight swift %}
func rotate(array: inout [Int], k: Int) {
  // Check for edge cases
  if k == 0 || array.count <= 1 {
    return // The resulting array is similar to the input array
  }

  // Calculate the effective number of rotations
  // -> "k % length" removes the abs(k) > n edge case
  // -> "(length +  k % length)" deals with the k < 0 edge case
  // -> if k > 0 the final "% length" removes the k > n edge case
  let length = array.count
  let rotations = (length +  k % length) % length


} {% endhighlight %}


## The "obvious" solution

Let's try to make the problem a little bit simpler. What does it really mean to rotate an array by one step to the right? It means to remove the last element of the array and insert it at the beginning.

{% highlight swift %}
original:
    [ 1 , 2 , 3 , 4 , 5 , 6 ]

remove last:
    [ 1 , 2 , 3 , 4 , 5 ] 6

insert it as the first element:
    [ 6 , 1 , 2 , 3 , 4 , 5 ] {% endhighlight %}

Converted into code:
{% highlight swift %}
// Right rotation
var array = [1,2,3,4,5,6]
let last = array.removeLast()
array.insert(last, at: 0)
{% endhighlight %}


At this moment, we know how to rotate the array to the right by one step. If we want to rotate the array by more than one step, we simply run the similar algorithm multiple times and we're done:

{% highlight swift %}
func rotate(array: inout [Int], k: Int) {
    // Check for edge cases
    if k == 0 || array.count <= 1 {
        return // The resulting array is similar to the input array
    }

    // Calculate the effective number of rotations
    // -> "k % length" removes the abs(k) > n edge case
    // -> "(length +  k % length)" deals with the k < 0 edge case
    // -> if k > 0 the final "% length" removes the k > n edge case
    let length = array.count
    let rotations = (length +  k % length) % length

    for _ in 0..<rotations {
        let last = array.removeLast()
        array.insert(last, at: 0)
    }
} {% endhighlight %}

{% highlight swift %}
// Objective-C version of the algorithm

+ (void) rotateArray: (NSMutableArray*)array by:(NSInteger)k {
    // Check for edge cases
    if (k == 0 || array.count <= 1) {
        return; // The resulting array is similar to the input array
    }

    // Count the effective number of rotations
    NSInteger length = array.count;
    NSInteger rotations = (length + k % length) % length;
    for (int i; i < rotations; i++) {
        id last = array[length - 1];
        [array removeLastObject]; // Why this method doesn't return the removed object? ¯\_(ツ)_/¯
        [array insertObject:last atIndex:0];
    }
} {% endhighlight %}

What is the **time complexity** of this solution? For every step we need to move all elements in the array. The worst case time complexity is *O*(n*k).

## The "better" solution

When ```k``` is small, the *O*(n\*k) time complexity of the previous solution isn't that bad. However, when ```k``` is getting close to ```n```, *O*(n\*k) is almost becoming *O*(n<sup>2</sup>). Let's see if we can do better.

Swift Array offers the ```reverse()``` function which can reverse the order of the elements in the array. The algorithm we're going to use can rotate the array by any number of steps only using this function. All we need are these 3 steps:

{% highlight swift %}
Original array                1 2 3 4 5 6
n = 6, k = 3

1. Reverse the whole array    6 5 4 3 2 1
2. Reverse first k numbers    4 5 6 3 2 1
3. Reverse last n-k numbers   4 5 6 1 2 3
{% endhighlight %}

Converting these steps to Swift is pretty straightforward:

{% highlight swift %}
func rotate(array: inout [Int], k: Int) {
    // Check for edge cases
    if k == 0 || array.count <= 1 {
        return // The resulting array is similar to the input array
    }

    // Calculate the effective number of rotations
    // -> "k % length" removes the abs(k) > n edge case
    // -> "(length +  k % length)" deals with the k < 0 edge case
    // -> if k > 0 the final "% length" removes the k > n edge case
    let length = array.count
    let rotations = (length + k % length) % length

    // 1. Reverse the whole array
    let reversed: Array = array.reversed()

    // 2. Reverse first k numbers
    let leftPart: Array = reversed[0..<rotations].reversed()

    // 3. Reverse last n-k numbers
    let rightPart: Array = reversed[rotations..<length].reversed()

    array = leftPart + rightPart
} {% endhighlight %}


{% highlight swift %}
// Objective-C version of the algorithm

+ (void) rotateArray: (NSMutableArray*)array by:(NSInteger)k {
  // Check for edge cases
  if (k == 0 || array.count <= 1) {
      return; // The resulting array is similar to the input array
  }

  // Count the effective number of rotations
  NSInteger length = array.count;
  NSInteger rotations = (length + k % length) % length;

  // 1. Reverse the whole array
  NSArray* reversed = [[array reverseObjectEnumerator] allObjects];

  // 2. Reverse first k numbers
  NSRange leftRange = NSMakeRange(0, rotations);
  NSArray* leftPart = [[[reversed objectsAtIndexes:[NSIndexSet indexSetWithIndexesInRange:leftRange]] reverseObjectEnumerator] allObjects];

  // 3. Reverse last n-k numbers
  NSRange rightRange = NSMakeRange(rotations, length - rotations);
  NSArray* rightPart = [[[reversed objectsAtIndexes:[NSIndexSet indexSetWithIndexesInRange:rightRange]] reverseObjectEnumerator] allObjects];

  // Replace objects in the original array
  [array replaceObjectsInRange:leftRange withObjectsFromArray:leftPart];
  [array replaceObjectsInRange:rightRange withObjectsFromArray:rightPart];
} {% endhighlight %}

Let's find the time complexity of this algorithm. The ```func reversed() -> Array``` function has complexity *O*(n). We use the function three times so the complexity is *O*(3n). Constants can be omitted so the final time complexity is *O*(n) 🎉.

Next time, we will invert some binary trees. Stay tuned.
