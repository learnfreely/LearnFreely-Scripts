# Pairing Algorithm

## Encoding courses

We first need a way to encode each course as a unique variable.
- we can use the first letter followed first number to denote level.
- To account for AP/IB, add an extra letter denoting the class type (A, I, or N for none)
- Chemistry 3200 $\rightarrow$ CN3
- IB English 4238 $\rightarrow$ EI4

## Parameters

- Subjects
  - Can leave out exeptions to simplify for now, assign score to each tutor so someone can easily choose the next highest score
- number of students
  - first priority to those with no student
  - next priority goes to tutor who wants the most students
- GPA
  - gets lower priority than the others, but still should have some amount of influence
 
## Representing each party

We would like to model the compatability of some tutor $T$ given a student $S$.

Let $S$ be the course code the student has chosen. For example:

$$S=\text{MN3}$$

Then $T$ can be a 4-tuple with the following elements

$$T=(\text{subjects, current-students, students-wanted, GPA})$$

We can reference an element of $T$ via superscript notation:

$$T^0=\text{subjects}$$

$$T^1=\text{current-students}$$

$$...$$

We can compute wether a tutor has any students, as well as the remaining slots they have using the number of students they currently have as well as how many they would like. 

In Python:

```
class Student:
    def __init__(self, subject: str):
        self.subject = subject

class Tutor:
    def __init__(self, subjects: List[str], current_students: int, students_wanted: int, GPA: float):
        self.subjects = subjects
        self.current_students = current_students
        self.students_wanted = students_wanted
        self.GPA = GPA

    def has_student(self):
        # easier to work with integers than boolean values
        return 0 if self.current_students == 0 else 1

    def remaining_spots(self):
        return (self.students_wanted - self.current_students)
```

## Creating the algorithm

We can now craeted a normalized weighted score for each tutor based on the compatability parameters. We can begin building a function $f$ to fulfil this task. $f$ will evolve as we add more and more functionality. 

Starting off, if the students subject is not an element of the list of subjects from the tutor, or if the tutor can no longer take students, we can automatically return 0. For this, we can use the indicator function which we will denote with square brackets: [ . ]

$$f(S_n, T_m)=[S_n \in T_m^0] \cdot [T_m^2-T_m^1 \neq 0] \cdot (\text{other parameters})$$

From there, first priority goes to the tutor with no students. That is, `if self.has_student() == 0`, that tutor gets increased priority.

In order to ensure that those without students are put first, we can do the same thing as the subject and create an indicator function. That being said, we'll give it a little leniency by allowing some non-zero score to be given even if the function evaluates to 0. 

$$f(S_n, T_m)=[S_n \in T_m^0] \cdot [T_m^2-T_m^1 \neq 0] \cdot \big([T_m^1 = 0] + \text{other parameters}\big)$$

Next priority is put on the tutor requesting the most students, regardless of how many they currently have. For example, a tutor with 3 students but requesting 6 will have higher priority than a tutor with 1 but requesting 2. 

This should return a value between 0 and 1, so we can use a scaled logistic function. 

$$\large g(x)=\frac{L}{1+e^{-k(x-x_0)}}$$

These are hyperparameters to mess with later, but letting $k=1$, $x_0=2$, and $L=1$ results in a mapping where $g(n) < g(n+1)$ for all $n$, but bounded by $0 < g(x) < L$. $k$ represents the "steepness" of the curve, but we'll leave it at 1 for now, and $x_0$ is the center point such that $g(x_0)=L/2$

Putting it all together, we can finish the definition of $g$ as such:

$$g(T_m)=\frac{1}{1+e^{-({T_m^2-T_m^1-2})}}$$

Which we can now incude in $f$:

$$f(S_n, T_m)= I(S_n, T_m) \cdot \big([T_m^1 = 0] + g(T_m) + \text{other parameters}\big)$$

Where $I$ represents the indicator coefficients.

Our final parameter is GPA, which will be given a lower weight. For now, we will have GPA influence the result by taking a value between 0 and 0.5.

Once again using a scaled logistic curve $j(x)$, we can set $k=2.5$, $x_0=3$, and $L=0.5$. This results in a curve upper bounded at $j(x) < 0.5$, centered at 3, and much steeper than $g(x)$ which had $k=1$. 

$$j(T_m)=\frac{1}{2(1+e^{-2.5(T_m^3-3)})}$$

Adding to $f$:

$$f(S_n, T_m)= I(S_n, T_m) \cdot \big([T_m^1 = 0] + g(T_m) + j(T_m)\big)$$

And written in its complete form:

$$f(S_n, T_m)=[S_n \in T_m^0] \cdot [T_m^2-T_m^1 \neq 0] \cdot \bigg([T_m^1 = 0] + \frac{1}{1+e^{T_m^1-T_m^2+2}} + \frac{1}{2+2e^{-2.5(T_m^3-3)})} \bigg)$$

## Evaluating all students and all tutors

There should be support for evaluation of multiple students. This causes a challenge since there would be clear favorability to the student evaluated first. To combat this, we can evaluate every student against every tutor before employing some method to allow for compromises between pairings if multiple students rank the same tutor highest. 

We can address this by wrapping the whole process into a binary relation $\mathrel{\hat{\mathrel{\mathcal R}}}$. 

Let $S$ and $T$ be the set of all students and tutors respectively. 

$$S=(S_0, S_1, \ldots, S_n)$$

$$T=(T_0, T_1, \ldots, T_m)$$

We can define our final function $\mathscr{F}$:

$$\mathscr{F}(S, T)=f(S \mathrel{\hat{\mathrel{\mathcal R}}} T)$$

Where $f$ acts on every element of $S \mathrel{\hat{\mathrel{\mathcal R}}} T$.

Doing it this way also makes the Python implementation easier

```
def compute_compatibility(S_i: Student, T_j: Tutor) -> float:
    # code to execute f(S_i, T_j)
    return compatibility_score

def total_compatibility(students: list[Student], tutors: list[Tutor]) -> list[list[float]]:
    total = []
    for i in range(len(students)):
        S_i = []
        for j in range(len(tutors)):
            S_i.append(compute_compatibility(students[i], tutors[j]))
        total.append(S_i)
    return total

'''
SAMPLE OUTPUT

total: [S_0, S_1, ..., S_n]

[
    [compute_compatability(students[0], tutors[0]),
     compute_compatability(students[0], tutors[1]),
     ...,
     compute_compatability(students[0], tutors[m]],

    [compute_compatability(students[1], tutors[0]),
     compute_compatability(students[1], tutors[1]),
     ...,
     compute_compatability(students[1], tutors[m]],

    ... ,

    [compute_compatability(students[n], tutors[0]),
     compute_compatability(students[n], tutors[1]),
     ...,
     compute_compatability(students[n], tutors[m]]
]
'''
```

We now have an array containing the compatability score for each student $S_i$ over all tutors. 

If multiple students have the same tutor as their highest score, those scores must be equal by definition of $\mathscr{F}$ as there is no dependence on any other student attributes besides subject.

$$\text{IN PROGRESS...}$$

Redefine student $S$ with new param: 

$$S={subject, GPA}$$

Edit $f$:

$$f(S_n, T_m)=[S_n^0 \in T_m^0] \cdot [T_m^2-T_m^1 \neq 0] \cdot \bigg([T_m^1 = 0] + \frac{1}{1+e^{T_m^1-T_m^2+2}} + \frac{[S_n^1 < T_m^3]}{2+2e^{-2.5(T_m^3 - S_n^1-2)})} \bigg)$$ 



$$[tutor GPA > student GPA] \cdot (some function(tutor GPA - student GPA))$$
