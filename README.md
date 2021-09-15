## Task.

The task is to build an AI agent that can solve any 9x9 sudoku in a reasonable amount of time.

## Approach.

The agent has been built using backtracking depth-first search with constraint propagation. The depth first search ensures that a solution will be found and the constraint propagation reduces the search space significantly which leads to improved performance. The sudoku problem can be formulated as a constraint satisfaction problem as follows. 
- There is one variable for each cell, giving a total of 81 variables. 
- Each unassigned variable has domain D = {1,..,9} and each variable for which a value has already been assigned, has a single value domain. 
- The constraints of a 9x9 sudoku are that the digits in each of the nine rows, columns and 3x3 boxes must be different, giving a total of 27 Alldiff constraints. 

Constraint propagation as per the constraints described above, indeed reduces significantly  the number of possible digits for each cell but this is not sufficient in terms of performance. Thus, in order to reduce the search space further and improve the performance I have also embedded there some additional inference strategies that were derived from, and in fact enforce, the constraints. There are many additional strategies that can be used to complement the "basic" constraint propagation in sudoku problems but without a search algorithm it cannot be ensured that the agent will be able to solve every sudoku problem. Based on that and evaluating the trade-off between the improvement in efficiency and the cost of coding up additional strategies in terms of time and effort, it was decided to implement two of them in conjunction with a backtracking depth-first search algorithm. The first strategy that was used refers to the situation where a possible value of a cell does not occur as a possible value in any other cell in the same row, column or 3x3 square (peer cell) and in such a case that value is assigned to the corresponding variable. The second strategy considers cases where two peer cells have the same domain and this is of length two. Then, those values are removed from the domains of the other peer cells. The constraint propagation has been used both as pre-processing in order to reduce the number of possible values for each cell before the search begins, as well as during the search to ensure consistency at each state and arrive faster either at an invalid state (because of an empty domain) or at a solution.

Once no more digits can be removed from the domain of any variable trough the constraint propagation, then the agent through the search algorithm picks a cell and assigns a value to that cell from the reduced domain. The choice of cell is determined by the most remaining values (MRV) heuristic. That is, the agent chooses the variable with the minimum number of possible values. In such a way, firstly the probability of picking a digit that leads to an invalid state is minimised and secondly in case of picking such a digit the cost is minimal. If there are more than one such variables, then the agent picks the one which leads to more constraint peers. The order in which the possible values from the chosen variable are tried is determined by the least constraint value (LCV) heuristic. That is, the values that eliminate less values from the domains of unassigned variables are picked first. In such a way since more possible values are left for the other variables, the values with higher probability to lead in a solution are tried first. After a choice of variable and value has been made, the domains of the peer variables get updated via constraint propagation. If the resulting state is not invalid or a solution (i.e there are still successors to explore), then the last added node to the search tree is expanded further by picking another variable and a new value from the domain of the new variable in the same way. It is thus evident that the algorithm always explores the deepest node in the current frontier until it arrives either at an invalid state or at a solution (depth first search). In case of arriving at an invalid state, the agent backtracks to the last valid state. That is, to the last state for which the domain is non-empty. It then picks the next value from that domain and repeats the process.


## Implementation.

 The purpose of this section is to provide some details on the implementation of the agent and mainly on the method `search(sudoku)`. The creation of sudoku states is conducted by creating objects of the class `sudoku_state`. This class is responsible for generally managing a sudoku state through its fields and methods. To create an object, a 9x9 numpy array should be initially provided through the constructor. Optionally, an array of possible values and an array of final values can be provided as well. The field `possible_values` holds the domain of each variable and the field `final_values` holds the values that have already been assigned to variables. Descriptions for the methods contained in this class can be found within the code as comments. Below we will see the implementation of the method `search(sudoku)` step by step.
<br/><br/>

``` Python

def search(sudoku):

    if type(sudoku) == tuple:
        state = sudoku_state(*sudoku)
             
    else:
        state = sudoku_state(sudoku)
              
    state.apply_constraints()  
  
    if state.is_invalid() == True: 
        return None
        
    if state.is_goal() == True:
        return state.final_values
```

The method accepts as input either a 9x9 numpy array that represents the sudoku or a tuple containing three 9x9 numpy arrays that represent respectively: the sudoku, the possible values and the final values for each cell. In any case, an object of the class sudoku_state gets created. Then, constraint propagation as pre-processing follows and the resulting state is checked for if it is invalid or a solution.
<br/><br/>


``` Python
    next_cell_candidates = [(len(state.possible_values[row][col]), row, col, state.possible_values[row][col]) for row in range(9) for col in range(9) if len(state.possible_values[row][col]) > 1]
    
    if len(next_cell_candidates) != 0:
        row,col,next_cell = pick_cell(state,next_cell_candidates) 
 ```
A list of candidates for the next cell to be picked is created and then the cell is picked by calling the method `pick_cell` with input the current state and the list of candidates. Below is a snapshot of that method.
<br/><br/>

``` Python
def pick_cell(state,next_cell_candidates):
    
    next_cell = min(next_cell_candidates, key = lambda t: t[0])
    
    # list of candidates with minimum number of possible values.
    min_len_candidates = [next_cell_candidates[i] for i in range(len(next_cell_candidates)) if next_cell_candidates[i][0] == next_cell[0]]

    candidates_and_occurances = []

    for cell in min_len_candidates:
        
        r = cell[1]
        c = cell[2]
        possible_values = cell[3]

        peers =  state.peers_per_unit(r,c)
        peers_unpacked = peers[0]+peers[1]+peers[2]
        total_occurances = 0
        occurances_list = []
        
        for value in possible_values:
            occurances = state.count_occurances(peers_unpacked,value)
            occurances_list.append(occurances)
            total_occurances += occurances
    
        candidates_and_occurances.append((cell,occurances,occurances_list))
    
    # among cells with minimum length choose the one with values that occur maximum times in peers.
    next_cell = max(candidates_and_occurances,key = lambda t: t[1])

    row = next_cell[0][1]
    col = next_cell[0][2]
    next_cell_ordered = [x for y, x in sorted(zip(next_cell[2],next_cell[0][3]), key = lambda t:t[1])]
    
    return row, col, next_cell_ordered
```
In this method, we pick the next cell to be explored and it is where the MRV and LCV heuristics are used. First, a list of cells with the minimum number of possible values is created and among those, the one whose values appear most times in peers' possible values is picked. After the cell is picked, the values in that cell are ordered based on the LCV heuristic.
<br/><br/>
Continuing with the method `search(sudoku)`, we have the following piece of code.


 ``` Python
        for num in next_cell:
            new_state = copy.deepcopy(state)
            new_state.update(row,col,num)
            new_state.possible_values[row][col] = [num]
            new_state.final_values[row][col] = num
            new_state.apply_constraints()
  
            if not new_state.is_invalid(): 
                deep_arr = search((new_state.final_values, new_state.possible_values, new_state.final_values))
                
                if deep_arr is not None:           
                    deep_state = sudoku_state(deep_arr,deep_arr, deep_arr)
                    
                    if deep_state.is_goal():
                        return deep_state.final_values
    return None

```
We iterate over the possible values of the picked cell. First, we create a copy of the state and we work on this new state. Then, we fix the new value and update the other cells via constraint propagation. If the resulting state is not invalid then we recursively call the method with input the newly created state which contains all the updates. Through the recursive call this new state is checked if it is invalid or a solution. If it is neither, the process continuous and the method calls itself until it arrives either at an invalid state or a solution. On arrival at an invalid state, 'None' is returned from the recursive call and the control flow continues from where it stopped in order to call itself. That is, it backtracks to the previous valid state and continuous with the for loop to try the next possible value for the cell that is currently under consideration. This continues until either every recursive call returns None in which case the whole method returns None and means that the sudoku has no solution or until it arrives at a goal state. In the latter case, deep_arr is not 'None' and a state is created from the values stored in `deep_state`. This state will be a goal state and the solution is returned. Notice that we don't iterate over every cell. Once a cell is picked initially, if none of the possible values leads to a solution then the method returns 'None and the sudoku has no solution. Also, notice that we create a (deep) copy of the state for each possible assignment otherwise the backtracking would not be possible as the actual state would have been modified each time. 