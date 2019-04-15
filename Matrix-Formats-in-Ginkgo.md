Matrix formats are used in Ginkgo to represent the system matrix and the vectors of a system. The following are the supported matrix formats with a short description and an example how the following matrix `A` might be stored:
```
    | 1 0 0 0 0 |
    | 0 0 5 0 0 |
A = | 4 0 0 0 3 |
    | 0 8 0 0 2 |
    | 0 0 0 7 0 |
```

### `gko::matrix::Dense`
This is the row-major storage dense matrix format. The matrix `A` would be stored in a single `value` array:
```
value = [ 1 0 0 0 0 0 0 5 0 0 4 0 0 0 3 0 8 0 0 2 0 0 0 7 0 ]
```

### `gko::matrix::Coo`
This is the Coordinate (COO) sparse matrix format.
Only stores the non-zero values, including the row- and column-indices corresponding to that value. Ideally, the entries are sorted first by row index, then by column index, but we only require that they are sorted by row index. Matrix `A` can be stored as follows:
```
val     = [ 1 5 4 3 8 2 7 ]
row_idx = [ 0 1 2 2 3 3 4 ]
col_idx = [ 0 2 0 4 1 4 3 ]
```

### `gko::matrix::Csr`
This is the Compressed Sparse Row (CSR) sparse matrix format.
It is basically the COO format sorted by row indices which compresses the row indices by only storing the beginning of each row and, in the end, the total number of non-zeros. This row-start array (here called `row_start`) always has `M + 1` entries for a `M x N` matrix. Matrix `A` would be stored as follows:
```
val       = [ 1 5 4 3 8 2 7 ]
row_start = [ 0 1 2 4 6 7 ]
col_idx   = [ 0 2 0 4 1 4 3 ]
```

### `gko::matrix::Ell`
This is the ELLPACK (ELL) sparse matrix format.
Before storing a matrix in ELL format, `max_nnz_per_row` needs to be determined, which is the maximum number of non-zeros in a row. Then, a `M x N` matrix is compressed to two `M x max_nnz_per_row` matrices. One to store the non-zero values (filled left to right, `V` in the example), and one to keep track of the column index of the corresponding value (`C` in the example). If a row of the new matrix is not fully populated, it is filled with zeros and the corresponding column indices can be arbitrary (here denoted with `X`). These two new matrices are then stored in column-major order. For matrix `A`, we also show the intermediate matrices `V` and `C`:
```
    | 1 0 |      | 0 X |
    | 5 0 |      | 2 X |
V = | 4 3 |  C = | 0 4 |
    | 8 2 |      | 1 4 |
    | 7 0 |      | 3 X |
storing column-major:
val = [ 1 5 4 8 7 0 0 3 2 0 ]
col = [ 0 2 0 1 3 X X 4 4 X ]
```

### `gko::matrix::Sellp`
This is the SELL-P sparse matrix format, which is based on the sliced ELLPACK representation.
Instead of storing the whole matrix in ELL format, groups of rows (__slices__ of the matrix) are stored in ELL format with the option to set a strice factor, which can be used to align the start of each slice, helping with cache alignment.
In the end, there are four arrays:
* `val`, which stores the non-zero values,
* `col_idx`, which stores the column index of the corresponding value,
* `slice_lengths`, which stores the maximum number of non-zeros per row of the corresponding slice, and
* `slice_sets`, which stores the prefix sum of `slice_lengths`, used to compute the beginning of slice `s` with: `slice_sets[s] * slice_size`. 
  
With a slice size of `2` and a stride factor of `1`, we would get the following result for matrix `A` (also including the intermediate step we showed for the ELL format):
```
V1 = | 1 |  C1 = | 0 |
     | 5 |       | 2 |
V2 = | 4 3 |  C2 = | 0 4 |
     | 8 2 |       | 1 4 |
V3 = | 7 |  C3 = | 3 |
     | 0 |       | X |

val           = [ 1 5 4 8 3 2 7 0 ]
col_idx       = [ 0 2 0 1 4 4 3 X ]
slice_sets    = [ 0 1 3 4 ]
slice_lengths = [ 1 2 1 ]
```

### `gko::matrix::Hybrid`
This is the hybrid matrix format that represents a matrix as a sum of an ELL and COO matrix.
At first, all values that can fit in the ELL format (in the example here, we chose `max_nnz_per_row = 1`) will be stored there and all entries that do not fit will be stored in COO format.
```
ell_val = [ 1 5 4 8 7 ]
ell_col = [ 0 2 0 1 3 ]
coo_val = [ 3 2 ]
coo_row = [ 2 3 ]
coo_col = [ 4 4 ]
```
