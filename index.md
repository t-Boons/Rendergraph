---
layout: default
title: Rendergraph
---
## Introduction
Rendering systems with high complexity are hard to manage which leads to bugs and unreadable code. That's why we have Rendergraphs to mitigate this issue, along with other optimizations that the use of a Rendergraph will bring to you if used in your rendering pipeline. In this blog I will be explaining how I designed and implemented a simple Rendergraph system into my DirectX 12 renderer called [Butterfly](https://google.com). This blog will be more focused on implementation details and roadblocks I hit along the way, reason being that I noticed many articles about this topic are often highlighting high level concepts instead of implementation details and code design. For this reason I chose to write this blog like this.

## What is a Rendergraph?
Rendergraphs aim to divide complex rendering work in to so called "Passes". Every pass contains a "step" in the rendering pipeline. For example writing to a GBuffer as pass 1. And calculating lighting using said GBuffer would be pass 2. By designing a system that allows easy sharing of dependencies (Pass 2 depends on Pass 1's work) we allow every pass to consist of a set of Inputs and Outputs. This makes it so every pass has their own set of data to work with rather than everything being data being all over the place.

*This is what a typical render pipeline would look like if all the dependencies are managed in one system.*
![[assets/TypicalRendering.png]]

*By using a rendergraph we define in and outputs for these render tasks and the pipeline becomes way cleaner.*
![[assets/RendergraphRendering.png]]

#### Optimization benefits.
Because we divide the work in to passes we have more control over when the gpu needs to be synced. These operations can be expensive and hard to manage in a complex rendering pipeline. By recording dependencies we know exactly which passes have dependencies and do gpu syncs accordingly, this also means the user does not have to worry about manually synchronizing anymore.

## What ingredients do we need.
To design a rendergraph system we need a few things:
* A way to define a pass's execution.
* A way to record any dependencies.
* A way to handle external dependencies.
* A way to create and manage temporary resources used only in the rendergraph.

### Pass execution.
There are several ways we can define a pass's execution. Like using the builder-pattern to append render-commands, overriding virtual functions inside of a class that inherits from Pass, or Lambda's. For my rendergraph I chose to work with Lambda's since it allows for more freedom than using the builder-pattern and less boilerplate than writing custom class implementations for every pass.

*This is what a pass in my rendergraph looks like:*
```cpp
	struct GBufferLight
	{
		BFRGTexture* PositionTexture;
		BFRGTexture* NormalTexture;
		BFRGTexture* DepthStencil;
		BFRGTexture* AlbedoTexture;
		BFRGTexture* OutputTexture;
	};

	builder.AddPass<GBufferLight>("Composite pass",
		[&](const GBufferLight& params, DX12CommandList& list)
		{
			// Lambda representing render commands.
		});
}
```

### Recording dependencies.
