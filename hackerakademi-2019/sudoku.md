# Sudoku

We're given a Python script with 24 different Sudoku puzzles. When you input
the solution to each grid, the script calculates a checksum of the grid and
returns another chracter of the flag. The challenge is to solve all 24
puzzles. This could be done by hand, but it's faster to use a solver.

I used a copy of the solver found at http://norvig.com/sudoku.html, with
some minor modifications to output the grids in the "bare" format, expected
by the challenge.

Running the solved grids through sudoku.py, gives us the flag.


## Flag

`FE{shall_we_play_a_game}`


---
Peter Tirsek, 2020-01-20
