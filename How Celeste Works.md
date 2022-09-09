[Celeste](https://www.lexaloffle.com/bbs/?tid=2145) is a really cool game made in 4 days that spurred the eventual creation of another title by the same name: [Celeste](https://store.steampowered.com/app/504230/Celeste/).

This game has really easy to read code, and is pretty fully featured as far as pico8 titles go. These notes break down some of the patterns used in the game that can be easily extended to any platformer game.

# Collectables
Ok, so you have things on the map that the player needs to 'collect' (read: interact with). So generally, you keep an array of those items and manage them in some code. Consider the below example for collecting money or something:

```lua

function _init()
	money={}
	player=m_player()
end

function _draw()
	-- draw the objects
	player:update()
	for o in all(money) do
		o:draw()
	end
end

function _update()
	-- update the objects
	player:draw()
	for o in all(money) do
		o:update()
	end
end

function m_money()
	add(money, {
		init=function(this)...end,
		update=function(this)...end,
		draw=function(this)...end,
	})
end

-- more code needed to actually drive interactions
```

So this setup is probably okay if you only have a few entities that interact with the player, but there are two drawbacks to this approach:

1. Adding a new object class, like `fruit` requires adding a new table to keep track of those objects and subsequent interactions
2. Suppose you have two objects that have common interactions with the player, but have a minor distinction (e.g. two types of powerups move, but in different ways). This setup to the game suggests that each should have logic in the update function of each object to handle this, but it can lead to a lot of similar logic that would be better off being shared.

### What Celeste Does For Organizing Game Objects

Celeste classis has a pretty clean way of handling this. It's basically inheiritance, with a base class, and each other object 'class' inheiriting all of the methods defined. Here's the relevant code from celeste:

```lua
function init_object(type,x,y)
    if type.if_not_fruit~=nil and got_fruit[1+level_index()] then
        return
    end

    local obj = {}

    obj.type = type
    obj.collideable=true
    obj.solids=true

    obj.spr = type.tile
    obj.flip = {x=false,y=false}

    obj.x = x
    obj.y = y
    obj.hitbox = { x=0,y=0,w=8,h=8 }
    
    obj.spd = {x=0,y=0}
    obj.rem = {x=0,y=0}

    obj.is_solid=function(ox,oy)
	    if oy>0 and not obj.check(platform,ox,0) and 
		obj.check(platform,ox,oy) then
			return true
		end
	
		return solid_at(obj.x+obj.hitbox.x+ox,obj.y+obj.hitbox.y+oy,obj.hitbox.w,obj.hitbox.h)
		or obj.check(fall_floor,ox,oy)
		or obj.check(fake_wall,ox,oy)
	end

    obj.is_ice=function(ox,oy)
        return ice_at(obj.x+obj.hitbox.x+ox,obj.y+obj.hitbox.y+oy,obj.hitbox.w,obj.hitbox.h)
	end

    obj.collide=function(type,ox,oy)
        local other
        for i=1,count(objects) do
            other=objects[i]
            if other ~=nil and other.type == type and other != obj and other.collideable and
                other.x+other.hitbox.x+other.hitbox.w > obj.x+obj.hitbox.x+ox and
                other.y+other.hitbox.y+other.hitbox.h > obj.y+obj.hitbox.y+oy and
                other.x+other.hitbox.x < obj.x+obj.hitbox.x+obj.hitbox.w+ox and
                other.y+other.hitbox.y < obj.y+obj.hitbox.y+obj.hitbox.h+oy then
                return other
            end
        end
        return nil
    end

    obj.check=function(type,ox,oy)
        return obj.collide(type,ox,oy) ~=nil
    end

    obj.move=function(ox,oy)
        local amount

        -- [x] get move amount
	    obj.rem.x += ox
        amount = flr(obj.rem.x + 0.5)
        obj.rem.x -= amount
        obj.move_x(amount,0)

        -- [y] get move amount
        obj.rem.y += oy
        amount = flr(obj.rem.y + 0.5)
        obj.rem.y -= amount
        obj.move_y(amount)
    end

    obj.move_x=function(amount,start)
        if obj.solids then
            local step = sign(amount)
            for i=start,abs(amount) do
                if not obj.is_solid(step,0) then
                    obj.x += step
                else
                    obj.spd.x = 0
                    obj.rem.x = 0
                    break
                end
            end
        else
            obj.x += amount
        end
    end

    obj.move_y=function(amount)
        if obj.solids then
            local step = sign(amount)

            for i=0,abs(amount) do
	            if not obj.is_solid(0,step) then
                    obj.y += step
                else
                    obj.spd.y = 0
                    obj.rem.y = 0
                    break
                end
            end
        else
            obj.y += amount
        end
    end

    add(objects,obj)
    if obj.type.init~=nil then
        obj.type.init(obj)
    end
    return obj
end
```

Now there is a lot going on here, but these are the bullet points:
1. The function takes a `type` varable that corresonds to a lua table with `update`, `draw`, and `init` functions. 
2. Some shared functionality is declared on the base table that is accessible in each of the `type` tables.
3. Functionality declared on the base table can be safely called on any other object in the project and used consistently.

Here's an example of an object defined in this system:

```lua
rare_coin={
	tile=12,
	init=function(this)
	end,
	update=function(this)
		local hit = this.collide(player,0,8)
		if hit~=nil then
			-- handle collision here!
			-- hit: player object
			-- this: rare_coin
		end
	end,
}
add(types, rare_coin)
```


### Generating Game Objects from Map Data

Notice how the above type has a tile property? This is used by Celeste to initialize objects from the map data. Basically, everytime a new level is loaded, all the tiles are iterated over and checked against existing types in the `types` table. When a tile matches a type, that type is passed into the above `init_object(type,x,y)` function with the tiles coordinates

From here, the game loop keeps track of the objects state and handles the interactions of the object with other objects in the game. It's a pretty neat system.