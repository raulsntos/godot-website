---
title: "Fixing tangents for accurate normal maps"
excerpt: "Correct tangents and bi-tangents are required for applying normal maps and newer effects such as the depth/parallax effect in the shader. Support for this has been spotty so we took the time out to fill in the gaps. These changes do have some consequences."
categories: ["progress-report"]
author: Bastiaan Olij
image: /storage/app/uploads/public/5bf/2ab/9f6/5bf2ab9f6f810514785264.png
date: 2018-11-21 12:19:14
---

While normal maps have been a staple of the Godot render engine for years, new capabilities of the render engine introduced in Godot 3.1 also require the generation of tangents and bi-tangents (often refered to as binormals in engines' documentation) to function correctly.

Tangents and bi-tangents are two vectors that together with the normal vector give enough orientation detail of a face to provide correct lighting. To save on space Godot uses the approach where only the tangents are store and bi-tangents are calculated by combining them with the normal vector. For this a direction flag is also included with each tangent.

Support for these so far has been spotty. The new CSG nodes lacked support for generating the tangents, there were orientation issues when importing Collada or glTF files, and the new primitive meshes reversed the direction of these.

We spent a great deal of time over the last week going through all these issues [and correcting them](https://github.com/godotengine/godot/pull/23760). All primitive meshes now generate correct tangents, all CSG nodes now support tangents, Sprite3D now supports tangents and the importers have been fixed.

![godot-tangents.jpg](/storage/app/uploads/public/5bf/54c/a51/5bf54ca51ea3d683171556.jpg)

This does mean that if you've worked around the existing issues by inverting your normal maps on one or both axes, or by using the "flip tangent" or "flip binormal" settings within materials, you may need to undo these changes when you upgrade to the current master build or the upcoming 3.1 alpha 3.

---

**Note**: on December 11th a further change was merged that ensures Godot can directly use normal maps, tangents and bi-tangents as generated by Blender and other modeling software following the same orientation rules.

*Header picture derived from [fracteed](https://twitter.com/fracteed)'s *[Parallax Heads](http://fracteed.com/godot.html)* demo (CC-BY 4.0).*
