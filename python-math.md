# Math and algorithmic problems solved in, or related to Python


## Question: [FFT division for fast polynomial division](https://stackoverflow.com/questions/44770632/fft-division-for-fast-polynomial-division/)

I'm trying to implement fast polynomial division using Fast Fourier Transform (fft).

Here is what I have got so far:

    from numpy.fft import fft, ifft
    def fft_div(C1, C2):
        # fft expects right-most for significant coefficients
        C1 = C1[::-1]
        C2 = C2[::-1]
        d = len(C1)+len(C2)-1
        c1 = fft(list(C1) + [0] * (d-len(C1)))
        c2 = fft(list(C2) + [0] * (d-len(C2)))
        res = list(ifft(c1-c2)[:d].real)
        # Reorder back to left-most and round to integer
        return [int(round(x)) for x in res[::-1]]

This works well for polynomials of same length, but if length is different then the result is wrong (I benchmark against [RosettaCode's][1] `extended_synthetic_division()` function):

    # Most signficant coefficient is left
    N = [1, -11, 0, -22, 1]
    D = [1, -3, 0, 1, 2]
    # OK case, same length for both polynomials
    fft_div(N, D)
    >> [0, 0, 0, 0, 0, -8, 0, -23, -1]
    extended_synthetic_division(N, D)
    >> ([1], [-8, 0, -23, -1])

    # NOT OK case, D is longer than N (also happens if shorter)
    D = [1, -3, 0, 1, 2, 20]
    fft_div(N, D)
    >> [0, 0, 0, 0, -1, 4, -11, -1, -24, -19]
    extended_synthetic_division(N, D)
    >> ([], [1, -11, 0, -22, 1])

What is weird is that it seems it's very close, but still a bit off. What did I do wrong? In other words: **how to generalize fast polynomial division (using FFT) to vectors of different sizes**.

Also bonus if you can tell me how to compute the division **quotient** (currently I only have the **remainder**).


  [1]: https://rosettacode.org/wiki/Polynomial_synthetic_division#Python


## Answer

Here's a direct implementation of a fast polynomial division algorithm found in these [lecture notes](http://web.cs.iastate.edu/~cs577/handouts/polydivide.pdf). 

The division is based on the fast/FFT multiplication of dividend with the divisor's reciprocal. My implementation below strictly follows the algorithm proven to have ``O(n*log(n))`` time complexity (for polynomials with degrees of the same order of magnitude), but it's written with emphasis on readability, not efficiency.

    from math import ceil, log
    from numpy.fft import fft, ifft

    def poly_deg(p):
        return len(p) - 1
    
    
    def poly_scale(p, n):
        """Multiply polynomial ``p(x)`` with ``x^n``.
        If n is negative, poly ``p(x)`` is divided with ``x^n``, and remainder is
        discarded (truncated division).
        """
        if n >= 0:
            return list(p) + [0] * n
        else:
            return list(p)[:n]
    
    
    def poly_scalar_mul(a, p):
        """Multiply polynomial ``p(x)`` with scalar (constant) ``a``."""
        return [a*pi for pi in p]
    
    
    def poly_extend(p, d):
        """Extend list ``p`` representing a polynomial ``p(x)`` to
        match polynomials of degree ``d-1``.
        """
        return [0] * (d-len(p)) + list(p)
    
    
    def poly_norm(p):
        """Normalize the polynomial ``p(x)`` to have a non-zero most significant
        coefficient.
        """
        for i,a in enumerate(p):
            if a != 0:
                return p[i:]
        return []
    
    
    def poly_add(u, v):
        """Add polynomials ``u(x)`` and ``v(x)``."""
        d = max(len(u), len(v))
        return [a+b for a,b in zip(poly_extend(u, d), poly_extend(v, d))]
    
    
    def poly_sub(u, v):
        """Subtract polynomials ``u(x)`` and ``v(x)``."""
        d = max(len(u), len(v))
        return poly_norm([a-b for a,b in zip(poly_extend(u, d), poly_extend(v, d))])
    
    
    def poly_mul(u, v):
        """Multiply polynomials ``u(x)`` and ``v(x)`` with FFT."""
        if not u or not v:
            return []
        d = poly_deg(u) + poly_deg(v) + 1
        U = fft(poly_extend(u, d)[::-1])
        V = fft(poly_extend(v, d)[::-1])
        res = list(ifft(U*V).real)
        return [int(round(x)) for x in res[::-1]]
    
    
    def poly_recip(p):
        """Calculate the reciprocal of polynomial ``p(x)`` with degree ``k-1``,
        defined as: ``x^(2k-2) / p(x)``, where ``k`` is a power of 2.
        """
        k = poly_deg(p) + 1
        assert k>0 and p[0] != 0 and 2**round(log(k,2)) == k
    
        if k == 1:
            return [1 / p[0]]
        
        q = poly_recip(p[:k/2])
        r = poly_sub(poly_scale(poly_scalar_mul(2, q), 3*k/2-2),
                     poly_mul(poly_mul(q, q), p))
    
        return poly_scale(r, -k+2)
    
    
    def poly_divmod(u, v):
        """Fast polynomial division ``u(x)`` / ``v(x)`` of polynomials with degrees
        m and n. Time complexity is ``O(n*log(n))`` if ``m`` is of the same order
        as ``n``.
        """
        if not u or not v:
            return []
        m = poly_deg(u)
        n = poly_deg(v)
        
        # ensure deg(v) is one less than some power of 2
        # by extending v -> ve, u -> ue (mult by x^nd)
        nd = int(2**ceil(log(n+1, 2))) - 1 - n
        ue = poly_scale(u, nd)
        ve = poly_scale(v, nd)
        me = m + nd
        ne = n + nd
    
        s = poly_recip(ve)
        q = poly_scale(poly_mul(ue, s), -2*ne)
    
        # handle the case when m>2n
        if me > 2*ne:
            # t = x^2n - s*v
            t = poly_sub(poly_scale([1], 2*ne), poly_mul(s, ve))
            q2, r2 = poly_divmod(poly_scale(poly_mul(ue, t), -2*ne), ve)
            q = poly_add(q, q2)
        
        # remainder, r = u - v*q
        r = poly_sub(u, poly_mul(v, q))
    
        return q, r

The ``poly_divmod(u, v)`` function returns a ``(quotient, remainder)`` tuple for polynomials ``u`` and ``v`` (like Python's standard ``divmod`` for numbers).

For example:

    >>> print poly_divmod([1,0,-1], [1,-1])
    ([1, 1], [])
    >>> print poly_divmod([3,-5,10,8], [1,2,-3])
    ([3, -11], [41, -25])
    >>> print poly_divmod([1, -11, 0, -22, 1], [1, -3, 0, 1, 2])
    ([1], [-8, 0, -23, -1])
    >>> print poly_divmod([1, -11, 0, -22, 1], [1, -3, 0, 1, 2, 20])
    ([], [1, -11, 0, -22, 1])

I.e:

- ``(x^2 - 1) / (x - 1) == x + 1``
- ``(2x^3 - 5x^2 + 10x + 8) / (x^2 + 2x -3) == 3x - 11``, with remainder ``41x - 25``
- etc. (Last two examples are yours.)


---


## Question: [Creating a binary tree through a loop](https://stackoverflow.com/q/44895195/404556)

I want to create a binary tree by iterating through a loop. I know how to write a very basic binary tree.

    class Tree(object):
        def __init__(self):
            self.left = None
            self.right = None
            self.data = None
    
    
    root = Tree()
    root.data = 75
    root.left = Tree()
    root.left.data = 95
    root.right = Tree()
    root.right.data = 64
    
    root.left.left = Tree()
    root.left.left.data = 32
    root.left.right = Tree()
    root.left.right.data = 93
    root.left.left = Tree()
    root.right.left.data = 32
    root.left.right = Tree()
    root.right.right.data = 93
    
    print(root.data)

This is tedious handtyping it, and if I were to have a list of numbers:

    list = [1,2,3,4,5,6,7]

and put it through a loop to create a binary tree in this order so:

       1 
     2   3
    4 5 6 7

How would I write that? and because I'm using this to calculate the sum of all the paths, how do you navigate/iterate through a binary tree:


## Answer

To build a full **binary tree** (*not necessarily* a binary search tree) from a list of node values, let's assume values in the list are ordered according to level-order traversal. So, a list from your question 

    values = [1,2,3,4,5,6,7]

will represent a tree:

       1 
     2   3
    4 5 6 7

Notice that the root value is at the position `0`, its left child is at the position `1*2-1=1`, the right child at `1*2=2`, etc. In general for a node in the list at index `i`, its left child is at position `(i+1)*2-1`, and the right child at `(i+1)*2`.

Now, we simply need to build the tree recursively, node by node, at each step creating the left and the right subtree.

To make it simpler, let's assume the list of `values` is global.

    class Node(object):
        def __init__(self, data=None, left=None, right=None):
            self.data = data
            self.left = left
            self.right = right
    
    def buildTree(i):
        if i < len(values):
            return Node(values[i], left=buildTree((i+1)*2-1), right=buildTree((i+1)*2))

    values = [1,2,3,4,5,6,7]
    tree = buildTree(0)

To print-out the tree, we can use the [preorder tree traversal](https://en.wikipedia.org/wiki/Tree_traversal#Pre-order):

    def preorder(node):
        if node is None:
            return
        yield node.data
        for v in preorder(node.left):
            yield v
        for v in preorder(node.right):
            yield v

Like this:

    >>> print list(preorder(tree))
    [1, 2, 4, 5, 3, 6, 7]
