We are trying to fix a bug in k-NN with uploading byte vectors to a remote store, so that in the remote store we can pull in the vectors into a GPU cluster and build an index on GPU.

The issue comes from Lucene in the MergedByteVectorValues. Here are details of the bug:

Remote indexing is a mechanism where vectors are uploaded from the data node, and a remote builder component is triggered to construct the vector index. Once the index is built, the data node stores it.
To accelerate the uploading process, we split the flat vectors into multiple logical partitions and use multipart uploading with several concurrent threads. (code) To determine the start offset of each partition, we skip a fixed number of vectors and then load the next one. (code)

// VectorValuesInputStream

private void setPosition(long n) throws IOException {
// Skip first N vectors
long bytesSkipped = 0;
int vectorsToSkip = (int) (n / bytesPerVector);
log.debug("Skipping {} bytes, {} vectors", n, vectorsToSkip);
int docId = knnVectorValues.docId();
while (docId != -1 && docId != DocIdSetIterator.NO_MORE_DOCS && vectorsToSkip > 0) {
docId = knnVectorValues.nextDoc();
bytesSkipped += bytesPerVector;
vectorsToSkip--;
}

    // After skipping the correct number of vectors, fill the buffer with the current vector
    reloadBuffer();

    // Advance to the correct position within the current vector
    long remainingBytes = n - bytesSkipped;
    if (remainingBytes > 0) {
        currentBuffer.position((int) remainingBytes);
    }
}

However, during the merge phase, we encountered an error while performing multipart uploading with byte flat vectors. The issue stems from Lucene’s MergedByteVectorValues, where incrementing the document ID does not advance the lastOrd field. (code)

// MergedByteVectorValues

@Override
public int nextDoc() throws IOException {
//
// Not increasing lastOrd in here!!
//
current = docIdMerger.next();
if (current == null) {
docId = NO_MORE_DOCS;
index = NO_MORE_DOCS;
} else {
docId = current.mappedDocID;
++index;
}
return docId;
}

As a result, when we try to load the next vector after skipping, the logic fails in a branch where lastOrd is always zero. (code) This problem doesn't occur in MergedFloatVectorValues, since lastOrd is properly incremented there. (code)
To resolve this, we need to fix the behavior in Lucene so that multipart uploads with byte vectors function correctly.

Description

MergedByteVectorValues represents a unified view of byte vector values from multiple underlying Lucene segments. It provides an iterator interface that allows advancing to a specific document ID. In this contract, if a user advances five times using the KnnVectorValues iterator and then loads the next byte[], it is expected to return the corresponding next vector.

However, this expected behavior does not hold for MergedByteVectorValues. After advancing N times and attempting to load the next byte[], an error consistently occurs. This inconsistency does not exist in MergedFloat32VectorValues, where the behavior is correct and reliable.

The root cause is that MergedByteVectorValues does not update the internal lastOrd field when advancing, unlike MergedFloat32VectorValues. As a result, when attempting to load the next vector, the code checks the current ord against lastOrd, which remains zero, causing an exception to be thrown.

lucene/lucene/core/src/java/org/apache/lucene/codecs/KnnVectorsWriter.java

Lines 450 to 461 in 7fc9fd3
@Override
public int nextDoc() throws IOException {
current = docIdMerger.next();
if (current == null) {
docId = NO_MORE_DOCS;
index = NO_MORE_DOCS;
} else {
docId = current.mappedDocID;
++index;
}
return docId;
}

To fix this, MergedByteVectorValues should be updated to increment lastOrd during advancement, mirroring the behavior of MergedFloat32VectorValues.

lucene/lucene/core/src/java/org/apache/lucene/codecs/KnnVectorsWriter.java

Lines 340 to 352 in 7fc9fd3
@Override
public int nextDoc() throws IOException {
current = docIdMerger.next();
if (current == null) {
docId = NO_MORE_DOCS;
index = NO_MORE_DOCS;
} else {
docId = current.mappedDocID;
++lastOrd;
++index;
}
return docId;
}

In OpenSearch, for fast vector index construction, we upload KNN vectors to a remote builder component and trigger the index build. To speed up uploads, vectors are logically partitioned, and multipart uploading is performed — a process that relies on the advance-then-load pattern. Due to lastOrd not being updated correctly in MergedByteVectorValues, this multipart upload mechanism currently fails, blocking fast uploads with byte vectors.

For more details use case in Opensearch, please refer to this : opensearch-project/k-NN#2803
Solution

The solution is straightforward: increment lastOrd during advancement, and ensure the ord check is made against lastOrd, not lastOrd + 1.

@Override
public byte[] vectorValue(int ord) throws IOException {
//  if (ord != lastOrd + 1) {
if (ord != lastOrd) { <----------------- It now compares against `lastOrd` not `lastOrd + 1`.
throw new IllegalStateException(
"only supports forward iteration: ord=" + ord + ", lastOrd=" + lastOrd);
} else {
lastOrd = ord;
}
return current.values.vectorValue(current.index());
}


@Override
public int nextDoc() throws IOException {
current = docIdMerger.next();
if (current == null) {
docId = NO_MORE_DOCS;
index = NO_MORE_DOCS;
} else {
docId = current.mappedDocID;
++lastOrd; <------------------------ This line should be added
++index;
}
return docId;
}

# FEATURE TASK

The Task is the following:
1. First add a unit test that fails without the fix in Lucene.
2. When the unit test has failed, log the failure to the bottom of this file in FAILING UNIT TEST DETAILS.
3. Add a git commit with the test.
4. Create the fix within the nextDoc() method.
5. Rerun the unit test and paste output below in SUCCESSFUL UNIT TEST DETAILS.

# FAILING UNIT TEST DETAILS

Test location: `lucene/core/src/test/org/apache/lucene/codecs/TestMergedByteVectorValues.java`

```
java.lang.AssertionError: Test failed during merge
    Caused by:
    java.lang.IllegalStateException: only supports forward iteration: ord=3, lastOrd=-1
        at org.apache.lucene.codecs.KnnVectorsWriter$MergedVectorValues$MergedByteVectorValues.vectorValue(KnnVectorsWriter.java:418)
        at org.apache.lucene.codecs.TestMergedByteVectorValues$1$1.mergeOneField(TestMergedByteVectorValues.java:138)
```

Reproduce with: `gradlew :lucene:core:test --tests "org.apache.lucene.codecs.TestMergedByteVectorValues.testSkipThenLoadByteVectorDuringMerge"`

# SUCCESSFUL UNIT TEST DETAILS

> Here, post output once the lastOrd behavior is fixed with the previously failing unit test now passing.
