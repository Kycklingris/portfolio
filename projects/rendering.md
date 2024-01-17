---
layout: project
title: Rendering
group_count: 1
url: https://github.com/Smorsoft/vct
url_text: Github
---
As already mentioned there isn't all that much to show currently. Most of the time has been spent on the voxelization and debug rendering of the voxels. The next step being major code cleanup, especially generalizing the render passes (probably using interfaces) and separating out the gltf stuff.

{% include preload_img.html
  src="/assets/projects/rendering/non_voxelized.png"
  aspect_ratio="calc(16/9)"
  alt="A picture showing the non voxelized sponza scene"
%}
{% include preload_img.html
  src="/assets/projects/rendering/voxelized.png"
  aspect_ratio="calc(16/9)"
  alt="A picture showing the voxelized sponza scene"
%}

# Voxelization
The voxelization is done in a compute shader running per triangle. The triangles dominant axis is calculated and then sorted from lowest to highest in the dominant axis. The three edges are voxelized using a algorithm based on [A Fast Voxel Traversal Algorithm for Ray Tracing](http://www.cse.yorku.ca/~amana/research/grid.pdf){:target="_blank"}.

It is currently voxelized into simple 3D texture, but considering that the voxelization is implemented using compute shaders it should not be a problem to change it into for example a sparse voxel octree.

Scanlines are calculated using the following function, the scanlines are voxelized using the same fast voxel traversal algorithm.
```wgsl
fn voxelize_interior(triangle: Triangle) {
	let next_plane = floor(triangle.vertices[0].grid_position[triangle.dom_axis] + 1.0);
	let max_plane = triangle.vertices[2].grid_position[triangle.dom_axis] - 0.5;

	var plane = next_plane + 0.5;

	let dir01 = triangle.vertices[0].grid_position - triangle.vertices[1].grid_position;
	let dir02 = triangle.vertices[0].grid_position - triangle.vertices[2].grid_position;
	let dir12 = triangle.vertices[1].grid_position - triangle.vertices[2].grid_position;

	let dom_length_01 = triangle.vertices[1].grid_position[triangle.dom_axis] - triangle.vertices[0].grid_position[triangle.dom_axis];
	let dom_length_02 = triangle.vertices[2].grid_position[triangle.dom_axis] - triangle.vertices[0].grid_position[triangle.dom_axis];
	let dom_length_12 = triangle.vertices[2].grid_position[triangle.dom_axis] - triangle.vertices[1].grid_position[triangle.dom_axis];

	while plane < max_plane {
		let t_02 = (plane - triangle.vertices[0].grid_position[triangle.dom_axis]) / dom_length_02;
		let p_02 = triangle.vertices[0].grid_position - (t_02 * dir02);

		var end: vec3<f32>;
		if triangle.vertices[1].grid_position[triangle.dom_axis] >= plane {
			let t_01 = (plane - triangle.vertices[0].grid_position[triangle.dom_axis]) / dom_length_01;
			let p_01 = triangle.vertices[0].grid_position - (t_01 * dir01);
			end = p_01;
		} else {
			let t_12 = (plane - triangle.vertices[1].grid_position[triangle.dom_axis]) / dom_length_12;
			let p_12 = triangle.vertices[1].grid_position - (t_12 * dir12);
			end = p_12;
		}

		voxelize_line(triangle, p_02, end);

		plane += 1.0;
	}
}
```

# Voxel Debug Rendering
This part is naive but good enough, it uses a two pass system of compute shaders. The first counts the amount of voxels in the 3D texture the cpu reads this and creates the index and vertex buffer. The second pass then filling the buffers with the actual voxel data. Is it fast? No. Has it occasionally lead to max buffer size errors? Yes. But it has been enough to verify that the voxelization runs without problems.

# Code Cleanup
As mentioned the next step is code cleanup, now, code quality was at the start of the project my main concern; however due to wanting something visual to show in this portfolio as well as wanting some written code to cleanup and plan around I decided to temporarily work on the voxelization. A big part of my current plan is the minification of boilerplate through the use of traits and macros, while simultaneously letting me use the rust type system for gpu buffers as well, I still haven't gotten particularly far with it but some sample code.
```rust
macro_rules! new_host_shareable {
	($type:ty, $wgsl_name:literal, $buffer_name:ident, $flags:expr) => {
		#[repr(transparent)]
		pub struct $buffer_name(::wgpu::Buffer);

		impl crate::Buffer for $buffer_name {
			type Source = $type;

			unsafe fn from_buffer(buffer: ::wgpu::Buffer) -> Self {
				::core::mem::transmute(buffer)
			}

			unsafe fn as_buffer(&self) -> &::wgpu::Buffer {
				::core::mem::transmute(self)
			}
		}

		impl crate::ToWGSL for $type {
			fn to_wgsl() -> crate::WGSL {
				crate::WGSL::Ty($wgsl_name.into())
			}
		}

		impl crate::HostShareable for $type {
			const REQUIRED_BUFFER_USAGE_FLAGS: ::wgpu::BufferUsages = $flags;
			type Buffer = $buffer_name;
		}
	};
	($type:ty, $wgsl_name:literal, $buffer_name:ident) => {
		new_host_shareable!(
			$type,
			$wgsl_name,
			$buffer_name,
			::wgpu::BufferUsages::empty()
		);
	};
}
```
```rust
pub trait HostShareable: Sized {
	const REQUIRED_BUFFER_USAGE_FLAGS: ::wgpu::BufferUsages;
	type Buffer: Buffer;

	unsafe fn as_bytes(&self) -> &[u8] {
		::core::slice::from_raw_parts(
			(self as *const Self) as *const u8,
			::core::mem::size_of::<Self>(),
		)
	}
}

pub trait Buffer: Sized {
	type Source: HostShareable;

	unsafe fn from_buffer(buffer: ::wgpu::Buffer) -> Self;

	unsafe fn as_buffer(&self) -> &::wgpu::Buffer;

	fn map_async(
		&self,
		mode: wgpu::MapMode,
		callback: impl FnOnce(Result<(), wgpu::BufferAsyncError>) + wgpu::WasmNotSend + 'static,
	) {
		let buffer = unsafe { self.as_buffer() };
		buffer.slice(..).map_async(mode, callback);
	}

	fn get_mapped_data<'a>(&'a self) -> &'a Self::Source {
		let buffer_view = unsafe { self.as_buffer() }.slice(..).get_mapped_range();
		unsafe { &*(buffer_view.as_ptr() as *const Self::Source) }
	}

	fn get_mapped_data_mut<'a>(&'a mut self) -> &'a mut Self::Source {
		let buffer_view = unsafe { self.as_buffer() }.slice(..).get_mapped_range_mut();
		unsafe { &mut *(buffer_view.as_ptr() as *mut Self::Source) }
	}
}

pub trait BufferArray: Sized {}

pub fn create_buffer<T: HostShareable + Sized>(
	device: &wgpu::Device,
	usage: ::wgpu::BufferUsages,
	mapped_at_creation: bool,
) -> T::Buffer {
	unsafe {
		T::Buffer::from_buffer(device.create_buffer(&wgpu::BufferDescriptor {
			label: None,
			size: std::mem::size_of::<T>() as u64,
			usage,
			mapped_at_creation,
		}))
	}
}

pub fn create_buffer_init<T: HostShareable + Sized>(
	device: &wgpu::Device,
	item: &T,
	usage: ::wgpu::BufferUsages,
) -> T::Buffer {
	use wgpu::util::DeviceExt;
	unsafe {
		T::Buffer::from_buffer(
			device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
				label: None,
				contents: item.as_bytes(),
				usage,
			}),
		)
	}
}
```