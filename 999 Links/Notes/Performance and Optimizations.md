## Identifying single rows efficiently

In order to significantly improve the compression ratio for the content column while still achieving fast retrieval of specific rows, 
pastila.nl is using **two hashes** (and a compound primary key) for identifying a specific row:

* a hash of the content, as discussed above, that is distinct for distinct data, and
* a [locality-sensitive hash](https://en.wikipedia.org/wiki/Locality-sensitive_hashing) (fingerprint) **that does not change on small changes** of data
