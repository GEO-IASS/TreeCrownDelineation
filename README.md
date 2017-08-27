# TreeCrownDelineation
## Evaluation Pipeline Documentation
### Directory structure and overall layout :

--src

--alignmodel

Has the random alignment module which outputs a csv file with the labels and
actual probability matrix

--classifymodel
Has the random classification module which outputs a csv file with the labels and
the probability matrix

--itcdmodel
Has one of the reference models we built over the last semester

--reportgen
Has the evaluation pipeline which evaluates the output for all three tasks

--report
Output folder where generated images and pdf are placed after evaluation

--template
Contains the html template which has the base document structure and
the css file

**Configuration** : The pipeline requires four paths to be mentioned in config.py i.e. datadir, indir
outdir and template_dir which should point to the dataset, files obtained from the participant,
empty directory and dir that contains the html file

**Running the pipeline** : The pipeline is part of the reportgen module and therefore one could
run the evaluation pipeline for all three tasks by running the __init__.py under the reportgen
folder.

**Extending (or) Swapping the pipeline** : The pipeline’s individual modules are swappable by
design and particularly it is easy to swap the rendering module from weasyprint by just reading
the template_vars dictionary which would hold all the variables which form the report.

**Projection** : The projection needs to be decided before the contest begins (Right now the code
converts everything to image coordinates to get a common measure). Once that is decided just
do the affine conversion being done in `generate_task_1`.

**Errors Possible**: Though checks are placed in almost all cases there might still be issues
during runtime and the following would serve as a quick hint to figure out potential problem
areas
1) **ShapefileException** - Ill formed shp file submitted by the participant, PyShp requires at
least one record per shape and hence we would require the user to add the plot
number on the shapes just to be safe.
2) **Divide by Zero** - Singularity encountered -> in most cases we might need to introduce a
very small value to make this go away since the numerator would also be zero anyways
i.e. as in tp/(tp+fp)
3) **Index Error / Key Error** - User submitted files that are unaligned i.e. #treeid and #stemid
should always match. Similarly if they predict for #species then each logit must contain
the exact number of float values
4) **Graphs are weirdly sized** - adjust the graph resolution property in reportgen module to fit
the graph.

Explanation for code is present as comments in most cases. Since task 1 has an elaborate
metric we have separate documentation

1. **class DelineationMetric** : This class exposes methods used for the assignment of ground truth to
predicted polygons and the calculation of scores representing the performance of the
delineation method.
2. **class MyPolygon** : This is a wrapper class written to perform all the functions related to the
polygons such as calculating intersection area, union area. Moreover it also keeps track of the
sum of intersection area of a polygons abject with other predicted polygons.

3. **performAssignment()** : This method takes the cost matrix(n X m) as the input. The cost here
represents the overlap area between ground truth polygon n i and the predicted polygon m i .
Using the Hungarian assignment algorithm each ground truth polygon is assigned to a single
predicted polygon so that the final Jaccard score is maximum.

4. **cleanUpData()**: This method removes the predicted geometric structures having less than 3
points. Therefore only polygons having more than 3 coordinate points will be accepted.
Moreover, polygons having area larger than a particular threshold are also not considered since
they represent spurious polygons.

5. **checkIfPolygonsIntersect()**: The following method checks if the predicted polygons are
intersecting among themselves. Since the overlap of predicted polygons is forbidden, we discard
the sum of area of overlap a polygon has with other polygon, in the calculation of Jaccard
similarity score, which might penalize the participants and eventually lead to low scores.

6. **formingPolygonsFromPoints()**: This method accepts a list of ITC and predicted polygons, and
convert the coordinate points for each polygons into a list of tuples which is accepted by the
shapely library.

7. **calculateHungarianAssignment()** : This is the most important method which needs to be called
before making calls to other functions of the class DelineationMetric. It accepts the list of
predicted and itc polygons as input, and controls the execution of the methods such as
cleanUpData, checkIfPolygonsIntersect, formingPolygonsFromPoints. It also forms the cost
matrix constituting the intersection areas and subsequently calls the performAssignment() to
initiate the assignment. Once the polygons have been properly assigned it eventually calculates
the jaccard score for the current plot and also add up this score to the final score for all the
plots. At the end it calls the setHistogramforRecall method to calculate the recall for that plot.

8. **getFinalJaccardScore()** : Returns the average jaccard score for all the plots. This score is a true
assessment of the performance of the crown delineation algorithm with higher scores being
more desirable outcome.

9. **calculateJaccardCoefficients()** : This method takes the current plot number, list of predicted
polygons, the assignedPoly list obtained from preformAssignment() and the cost matrix
generated in calculateHungarianAssignment(). For each of the mapped pair of polygons in the
assignedPoly list, truepositive , false positives and false negatives is calculated on a plot level as
well across all the plots. Finally a dictionary is formed with plot number and polygon number as
the key and jaccard score and polygon points as the values.

10. **getHistogramForRecall ()**: This method computes the histogram with the recall value between 0
and 1 divided into 10 bins as the x axis and the count of polygons having a certain recall values
on the Y axis. This gives the participants an intuition as to how their algorithm is performing.

11. **getTopPolygons ()** - Returns the top 4 best and worst polygons predicted across all the plots. The
dictionary returned has the (plot number, polygon number) as the key where as the similarity
score and similarity score and polygon points are the values.

12. **getConfusionMatrix ()** : Returns a tuple representing the total truepositive , false positives and
false negatives in the form of area across all the plots.

13. **getConfusionMatrixOnPlotLevel ()** : Acts similar to the getConfusionMatrix() method but instead
shows results on a per plot basis.
