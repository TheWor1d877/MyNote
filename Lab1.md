## 1. 
Draw the inverted index for the following document collection:
Doc 1 new home sales top forecasts
Doc 2 home sales rise in july
Doc 3 increase in home sales in july
Doc 4 july new home sales rise
#### answer
| Term             | Postings List (Document IDs) |
|------------------|------------------------------|
| new              | [1, 4]                       |
| home             | [1, 2, 3, 4]                 |
| sales            | [1, 2, 3, 4]                 |
| top              | [1]                          |
| forecasts        | [1]                          |
| rise             | [2, 4]                       |
| in               | [2, 3]                       |
| july             | [2, 3, 4]                    |
| increase         | [3]                          |

## 2 
Consider these documents:
Doc 1 breakthrough drug for schizophrenia
Doc 2 new schizophrenia drug
Doc 3 new approach for treatment of schizophrenia
Doc 4 new hopes for schizophrenia patients
(a) Draw the term-document incidence matrix for this document collection.
(b) Draw the inverted index representation for this collection.
(c) What are the returned results for these queries:
i. schizophrenia AND drug
ii. for AND NOT (drug OR approach)
#### answer
(a):

| Term              | Doc1 | Doc2 | Doc3 | Doc4 |
|-------------------|------|------|------|------|
| approach          | 0    | 0    | 1    | 0    |
| breakthrough      | 1    | 0    | 0    | 0    |
| drug              | 1    | 1    | 0    | 0    |
| for               | 1    | 0    | 1    | 1    |
| hopes             | 0    | 0    | 0    | 1    |
| new               | 0    | 1    | 1    | 1    |
| of                | 0    | 0    | 1    | 0    |
| patients          | 0    | 0    | 0    | 1    |
| schizophrenia     | 1    | 1    | 1    | 1    |
| treatment         | 0    | 0    | 1    | 0    |

(b):

| Term              | Postings List (Document IDs) |
|-------------------|------------------------------|
| approach          | [3]                          |
| breakthrough      | [1]                          |
| drug              | [1, 2]                       |
| for               | [1, 3, 4]                    |
| hopes             | [4]                          |
| new               | [2, 3, 4]                    |
| of                | [3]                          |
| patients          | [4]                          |
| schizophrenia     | [1, 2, 3, 4]                 |
| treatment         | [3]                          |

(c)
i. schizophrenia AND drug
schizophrenia: [1, 2, 3, 4]
drug: [1, 2]
Intersection: [1, 2]
ii. for AND NOT(drug OR approach)
for: [1, 3, 4]
drug OR approach: [1, 2, 3]
NOT(drug OR approach): Documents not in [1, 2, 3] → [4]
Intersection: [1, 3, 4] ∩ [4] = [4]

## 3
Implement in Python the intersect of two postings algorithm from classes. Assume the posting are lists of integers.

#### answer
```python
def intersect_postings(list1, list2):
    """
    Returns the intersection of two posting lists (sorted integer lists).
    Time complexity: O(m + n)
    """
    result = []
    i, j = 0, 0
    while i < len(list1) and j < len(list2):
        if list1[i] == list2[j]:
            result.append(list1[i])
            i += 1
            j += 1
        elif list1[i] < list2[j]:
            i += 1
        else:
            j += 1
    return result
```

## 4
Implement in Python the intersect of n terms algorithm from classes. The function should
receive a list of terms and an inverted index. Assume the inverted index is represented by a
dictionary where the keys terms and the values are pairs with the document frequency and
the postings (that is, list of document ids). An example follows:
{
’ forecasts ’ : (1 , [ 1 ] ) ,
’ home ’ : ( 4 , [ 1 , 2 , 3 , 4 ] ) ,
’ in ’ : ( 2 , [ 2 , 3 ] ) ,
’ increases ’ : (1 , [ 3 ] ) ,
’ july ’ : (3 , [2 , 3 , 4 ] ) ,
’ new ’ : ( 1 , [ 1 ] ) ,
’ rise ’ : (2 , [2 , 4]) ,
’ sales ’ : (4 , [1 , 2 , 3 , 4]) ,
’ top ’ : ( 1 , [ 1 ] )
}
The first element in the pair is the document frequency and the second is the posting.
#### answer
```bash
from collections import defaultdict

def threshold_query(postings_dict, query_terms, k):
    """
    postings_dict: dict mapping term -> sorted list of docIDs
    query_terms: list of query terms, e.g., ['A', 'B', 'C']
    k: minimum number of query terms a document must contain
    Returns: sorted list of qualifying docIDs
    """
    doc_count = defaultdict(int)
    
    for term in query_terms:
        if term in postings_dict:
            for doc_id in postings_dict[term]:
                doc_count[doc_id] += 1
    
    result = [doc_id for doc_id, count in doc_count.items() if count >= k]
    result.sort()
    return result
```

## 5
Recommend a query processing order for
(tangerine OR trees) AND (marmalade OR skies) AND (kaleidoscope OR eyes)
given the following postings list sizes:
Term                        Postings size
eyes                         213312
kaleidoscope          87009
marmalade             107913
skies                        271658
tangerine                46653
trees                        316812
Justify your answer.
#### answer
(tangerine OR trees): ≤ 46,653 + 316,812 = 363,465
(marmalade OR skies): ≤ 107,913 + 271,658 = 379,571
(kaleidoscope OR eyes): ≤ 87,009 + 213,312 = 300,321
Since AND operations benefit from smaller inputs (the result cannot exceed the size of either operand), we should start with the smallest intermediate result.
Thus, evaluate in this order:
First: (kaleidoscope OR eyes)
Then: AND with (tangerine OR trees)
Finally: AND with (marmalade OR skies)

## 6
Write a postings merge Python function that evaluates the query x AND (NOT y) efficiently.

#### answer
```python
def and_not_merge(postings_x, postings_y):
    """
    Efficiently computes the result of the Boolean query: x AND (NOT y)
    
    Parameters:
        postings_x (list of int): Sorted list of docIDs containing term x.
        postings_y (list of int): Sorted list of docIDs containing term y.
    
    Returns:
        list of int: Sorted list of docIDs that contain x but not y.
    """
    result= []
    i,j = 0
    len_x = len(postings_x)
    len_y = len(postings_y)

    while i < len_x:
        word_x = postings_x[i]
        while j < len_y and postings_y[j] < word_x:
            j+=1

        if postings_y[j] == word_x:
            i+=1
        else:
            result.append(word_x)
            i+=1
    return result
```

## 7
Try using the Boolean search features on a couple of major web search engines. For instance,
choose a word, such as burglar, and submit the queries (i) burglar, (ii) burglar AND burglar,
and (iii) burglar OR burglar. Look at the estimated number of results and top hits. Do they
make sense in terms of Boolean logic? Often they haven’t for major search engines. Can
you make sense of what is going on? What about if you try different words? For example,
query for (i) knight, (ii) conquer, and then (iii) knight OR conquer. What bound should
the number of results from the first two queries place on the third query? Is this bound
observed?
#### answer

![[Attachments/Pasted image 20260508135939.png]]


![[Attachments/Pasted image 20260508135959.png]]

![[Attachments/Pasted image 20260508140017.png]]
burglar: 22,500
burglar AND burglar: 74,400 (much higher!)
burglar OR burglar: 22,500 (as expected)
This shows Bing does not strictly follow Boolean logic for redundant terms.
AND may trigger a different ranking or query expansion (e.g., including synonyms or related pages), inflating results;
Thus, major search engines do not reliably implement true Boolean retrieval, especially with tautological queries.

![[Attachments/Pasted image 20260508135405.png]]
![[Attachments/Pasted image 20260508135434.png]]
![[Attachments/Pasted image 20260508135457.png]]
Boolean logic requires that knight OR conquer return at least as many results as the larger of the two individual queries (≥157,000). The Bing result of 190,000 satisfies this lower bound, showing that Bing respects Boolean OR when written in uppercase.