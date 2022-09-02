
Collision detection get's complicated pretty fast once you move past bounding boxes over sprites. Here are some complex scenarios just to get the brain juices flowing:

- Colliding sprite with dynamically rendered content (not sprite).
- Colliding oddly shaped sprites (think walking up a slope)
- Colliding objects with each other conditionally

Generalized Collision Detection for all of these scenarios seems pretty hard, but maybe you can do a combo of a few different methods?

PICO-8 treats sprites on the plane differently than you might expect:
1. positive y is down
2. the sprite location is the top-left corner of the sprite.

With this in mind, we can define normal bounding box collisions for sprites $S1$ and $S2$. If all of the following inequalities are true, we have a collision:
$$\begin{align} S1_x < S2_x + S2_{width} \\ 
S1_y < S2_y + S2_{height} \\ 
S1_y + S1_{height} > S2_y \\ 
S1_x + S1_{width} > S2_x \\ \end{align}$$
These equations are basically saying that for a collision to occur, one of the starting coordinates (x or y) of a sprite must be less than the starting coordinate + width of another. These can be rewritten to use subtraction as well, if someone so chooses, just subtract the added width or height from both sides of each inequality to get:
$$ \begin{align} S1_x - S2_{width} < S2_x \\ 
S1_y - S2_{height} < S2_y \\ 
S1_y > S2_y - S1_{height} \\ 
S1_x > S2_x - S1_{width} \\ \end{align} $$

It doesn't really matter which set of equations you use, they both describe the conditions under which two bounding boxes collide. Using the first set, we can write a collision function in lua as:

```lua
-- Collision between two sprites
function collide(s1, s2)
	return (
		s1.x < s2.x + s2.w &&
		s1.y < s2.y + s2.h &&
		s2.x < s1.x + s2.w &&
		s2.y < s1.y + s1.h 
	)
end
```

The key here is that this:

**We're only checking to see if one bounding box corner is contained within another bounding box**

This is great!... but as I've found, also difficult to use in an update function for the game loop. What we can do instead, is check each entity's location in the map to see if they are near any bounding boxes (collideable sprites), and make use of PICO-8's flag system to keep track of which sprites can be interacted with. 

This can be implemented by keeping track of $dx$ & $dy$ variables on each entity and using those to calculate the 'potential position', then running collision detection against that. If the potential position of a dx shift is not valid (collides), then set $dx=0$. Same process for $dy$

[This video explains the logic with drawings, pretty well](https://www.youtube.com/watch?v=Gs0XFViFxFs)

## Small Objects Moving Fast

This collision detection method doesn't work for entities that are sufficiently small, fast, or both. Small/fast entities tend to tunnel through collision blocks when they have a $dx$ or $dy$ value greate enough that their "potential position" is on the other side of a collision block. If entities are moving that fast, you can check a subset of the potential positions along the way. So instead of checkout positions $(x,y)$ and $(x+dx,y)$ for instance, you would check positions:

$(x,y), (x+1*dx/n,y), (x+2*dx/n,y)...(x+n*dx/n,y)$

Where $n$ is sufficiently large (but not too large) to capture all intermediate tiles.