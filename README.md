LINQ Raytracer
==============

A LINQified raytracer implementation in C#.  This sample is from a 2007 [blog post](http://blogs.msdn.com/b/lukeh/archive/2007/10/01/taking-linq-to-objects-to-extremes-a-fully-linqified-raytracer.aspx) which has additional details.

## Running

    csc LINQRayTracer.cs
    LINQRayTracer.exe

## Raytracer LINQ query

The majority of the raytracing implementation is in a [single LINQ query](https://github.com/lukehoban/LINQ-raytracer/blob/master/LINQRayTracer.cs#L48).

```csharp
var pixelsQuery =
  from y in Enumerable.Range(0, screenHeight)
  let recenterY = -(y - (screenHeight / 2.0)) / (2.0 * screenHeight)
  select from x in Enumerable.Range(0, screenWidth)
    let recenterX = (x - (screenWidth / 2.0)) / (2.0 * screenWidth)
    let point =
      Vector.Norm(Vector.Plus(scene.Camera.Forward,
                  Vector.Plus(Vector.Times(recenterX, scene.Camera.Right),
                  Vector.Times(recenterY, scene.Camera.Up))))
    let ray = new Ray() { Start = scene.Camera.Pos, Dir = point }
    let computeTraceRay = (Func<Func<TraceRayArgs, Color>, Func<TraceRayArgs, Color>>)
      (f => traceRayArgs => (
        from isect in
          from thing in traceRayArgs.Scene.Things
          select thing.Intersect(traceRayArgs.Ray)
        where isect != null
        orderby isect.Dist
        let d = isect.Ray.Dir
        let pos = Vector.Plus(Vector.Times(isect.Dist, isect.Ray.Dir), isect.Ray.Start)
        let normal = isect.Thing.Normal(pos)
        let reflectDir = Vector.Minus(d, Vector.Times(2 * Vector.Dot(normal, d), normal))
        let naturalColors =
          from light in traceRayArgs.Scene.Lights
          let ldis = Vector.Minus(light.Pos, pos)
          let livec = Vector.Norm(ldis)
          let testRay = new Ray() { Start = pos, Dir = livec }
          let testIsects =
            from inter in
              from thing in traceRayArgs.Scene.Things
              select thing.Intersect(testRay)
            where inter != null
            orderby inter.Dist
            select inter
          let testIsect = testIsects.FirstOrDefault()
          let neatIsect = testIsect == null ? 0 : testIsect.Dist
          let isInShadow = !((neatIsect > Vector.Mag(ldis)) || (neatIsect == 0))
          where !isInShadow
          let illum = Vector.Dot(livec, normal)
          let lcolor = illum > 0 ? Color.Times(illum, light.Color) : Color.Make(0, 0, 0)
          let specular = Vector.Dot(livec, Vector.Norm(reflectDir))
          let scolor = specular > 0
                         ? Color.Times(Math.Pow(specular, isect.Thing.Surface.Roughness),
                                       light.Color)
                         : Color.Make(0, 0, 0)
          select Color.Plus(Color.Times(isect.Thing.Surface.Diffuse(pos), lcolor),
                            Color.Times(isect.Thing.Surface.Specular(pos), scolor))
          let reflectPos = Vector.Plus(pos, Vector.Times(.001, reflectDir))
          let reflectColor = traceRayArgs.Depth >= MaxDepth
                              ? Color.Make(.5, .5, .5)
                              : Color.Times(isect.Thing.Surface.Reflect(reflectPos),
                                            f(new TraceRayArgs(new Ray()
                                            {
                                              Start = reflectPos,
                                              Dir = reflectDir
                                            },
                                            traceRayArgs.Scene,
                                            traceRayArgs.Depth + 1)))
          select naturalColors.Aggregate(reflectColor,
                                        (color, natColor) => Color.Plus(color, natColor))
        ).DefaultIfEmpty(Color.Background).First())
    let traceRay = Y(computeTraceRay)
    select new { X = x, Y = y, Color = traceRay(new TraceRayArgs(ray, scene, 0)) };

foreach (var row in pixelsQuery)
  foreach (var pixel in row)
    setPixel(pixel.X, pixel.Y, pixel.Color.ToDrawingColor());
```
