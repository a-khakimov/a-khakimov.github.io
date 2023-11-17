---
layout: post
title: 'QCoolLine : bfs algorythm'
date: 2018-05-20 11:42:00 +0000
categories: [Algorythms, Qt, C/C++]
tags: [qt, algorythm, bfs, c/c++, алгоритмы]
---

One day it happened that I had a task to write a small program using [Qt Creator](https://www.qt.io/), which would build the shortest path on the stage between two points. In general, this program is useless and it is nowhere to apply, but for me was useful experience of writing this program. To find the shortest path I decided to use the [wide search](https://en.wikipedia.org/wiki/Breadth-first_search) algorithm.

##  Stack

* C/C++
* Qt Creator

## Realisation of BFS algorithm

The following points should be taken into account when implementing **BFS**:

* implementation should be in a separate class
* when creating its object, the field size must be set
* availability of the **setRect** method for obtain the coordinates of obstacles and place them on the field
* availability of a method to calculate the shortest path that takes the start and end points

In general, the algorithm works as if we are starting a wave from the **A** point, which propagates along the field and as soon as it reaches the **B** point, the shortest path is calculated by the passed wave.
This is more clearly seen in the animated image.


![](assets/img/qcoolline/Lee.gif){: w="400" : .normal : .shadow }


Having understood how the algorithm works, you can write its implementation:

* first, you need to check whether the point is located on the obstacle
* if not, put a point in the queue
* start looking around at one cell *(I decided that it is enough to look up, down, left and right)*
* if the cell is empty, then put it in the queue to further look at the sides of the position of this point
* and so on until we get to the point **B**

```cpp
bool LeeAlgorythm::CustomLee(	const QPoint &beginPoint, 
				const QPoint &endPoint) {
    if(getGridVal(beginPoint) || getGridVal(endPoint))
        return false;
    QQueue<QPoint> wave;
    wave.push_back(beginPoint);
    setGridVal(beginPoint, mark);
    while(!wave.empty()) {
        for(size_t i = 0; i < 4; ++i) {
            QPoint tmpPoint( wave.front().x() + hx[i], 
			     wave.front().y() + hy[i]);
            if(	tmpPoint.x() >= 0 && 
		tmpPoint.x() < W && tmpPoint.y() >= 0 && 
		tmpPoint.y() < H) {
                if(getGridVal(tmpPoint) == 0 ) {
                    wave.push_back(tmpPoint);
                    setGridVal(tmpPoint, getGridVal(wave.front()) - 1);
                    if(	tmpPoint.x() == endPoint.x() && 
			tmpPoint.y() == endPoint.y()) {
                        this->path = GetPath(tmpPoint);
                        return true;
                    }
                }
            }
        }
        wave.pop_front();
    }
    return false;
}
```

As a result, it turns out that the wave will spread across the field. If the cell is our point **B**, then call the **GetPath** method, which will get the shortest path along the wave propagation circles.

```cpp
QVector<QPoint> LeeAlgorythm::GetPath(const QPoint &endPoint) {
    QVector<QPoint> path;
    path.push_back(endPoint);
    int max_val = getGridVal(path.back());
    while(max_val < -1) {
        for(size_t i = 0; i < 4; ++i) {
            QPoint tmpPoint = { path.back().x() + hx[i], 
				path.back().y() + hy[i] };
            if(	tmpPoint.x() >= 0 && 
		tmpPoint.x() < W && 
		tmpPoint.y() >= 0 && 
		tmpPoint.y() < H) {
                if(max_val < getGridVal(tmpPoint) && 
		   getGridVal(tmpPoint) < 0 ) {
                    max_val = getGridVal(tmpPoint);
                    path.push_back(tmpPoint);
                }
            }
        }
    }
    return path;
}
```

## Graphic part

I used [QGraphicsScene](http://doc.qt.io/qt-5/qgraphicsscene.html) for graphics. It allows you to draw primitive things like lines and points, place widgets on the scene, and work with incoming events. At the beginning of the scene drawn obstacles in the form of random rectangles. Obstacles can be re-generated. Well, then the user is to specify two points and get the result. I agree that the following code does not look very nice, but it does the following:

* when you click the mouse to select a point we just remembered the position
* and when you click again in another place, will be called the method **CustomLee**, which calculates the shortest distance and then in the **for** cycle the line is drawn in the scene by the **scene->addEllipse** method.


```cpp
void QCoolLine::mousePressEvent(QMouseEvent *mouse_event) {
    QPoint p = mouse_event->pos();
    if(p.y() < sceneY_sz) {
        switch (mouse_es) {
        case set_first_point:
            point1 = p;
            mouse_es = set_second_point;
            break;

        case set_second_point:
            point2 = p;
            leeAlg->grid = grid;
            if(leeAlg->CustomLee(point1, point2))
                for(int i = 0; i < leeAlg->path.size(); i++)
                    scene->addEllipse(	leeAlg->path.at(i).x(), 
					leeAlg->path.at(i).y(), 1, 1, 
					QPen(Qt::red));
            mouse_es = set_first_point;
            leeAlg->path.clear();
            break;
        }
    }
}
```

The result is something like this.

![Image](assets/img/qcoolline/screen.png){: w="500" }

The source code for this program is available [here](https://github.com/techlinked/QCoolLine).

----------------
