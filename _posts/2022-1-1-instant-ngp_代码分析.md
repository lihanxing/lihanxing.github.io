---
layout: post
title:  "instant-ngp"
date:   2022-07-7
categories: 三维重建
---
* content
{:toc}

---

render_nerf()函数分析

```
void Testbed::render_nerf(CudaRenderBuffer& render_buffer, const Vector2i& max_res, const Vector2f& focal_length, const Matrix<float, 3, 4>& camera_matrix0, const Matrix<float, 3, 4>& camera_matrix1, const Vector2f& screen_center, cudaStream_t stream) 
{
	// Reserve the memory for max-res rendering to prevent stuttering
	m_nerf.tracer.enlarge(max_res.x() * max_res.y(), m_network->padded_output_width());

	float plane_z = m_slice_plane_z + m_scale;
	if (m_render_mode == ERenderMode::Slice) {
		plane_z = -plane_z;
	}

	ERenderMode render_mode = m_visualized_dimension > -1 ? ERenderMode::EncodingVis : m_render_mode;

	m_nerf.tracer.init_rays_from_camera(
		render_buffer.spp(),
		m_network->padded_output_width(),
		render_buffer.resolution(),
		focal_length,
		camera_matrix0,
		camera_matrix1,
		screen_center,
		m_snap_to_pixel_centers,
		m_render_aabb,
		plane_z,
		m_dof,
		m_nerf.render_with_camera_distortion ? m_nerf.training.dataset.camera_distortion : CameraDistortion{},
		m_envmap.envmap->params_inference(),
		m_envmap.resolution,
		m_nerf.render_with_camera_distortion ? m_distortion.map->params_inference() : nullptr,
		m_distortion.resolution,
		render_buffer.frame_buffer(),
		m_nerf.density_grid_bitfield.data(),
		m_nerf.show_accel,
		m_nerf.cone_angle_constant,
		render_mode,
		stream
	);

	uint32_t n_hit;
	if (m_render_mode == ERenderMode::Slice) {
		n_hit = m_nerf.tracer.n_rays_initialized();
	} else {
		float depth_scale = 1.f/m_nerf.training.dataset.scale;
		n_hit = m_nerf.tracer.trace(
			*m_nerf_network,
			m_render_aabb,
			m_aabb,
			m_nerf.training.n_images,
			m_nerf.training.transforms.data(),
			focal_length,
			m_nerf.cone_angle_constant,
			m_nerf.density_grid_bitfield.data(),
			render_mode, camera_matrix1, depth_scale, m_visualized_layer, m_visualized_dimension,
			m_nerf.rgb_activation, m_nerf.density_activation, m_nerf.show_accel, m_nerf.rendering_min_alpha,
			stream
		);
	}
	RaysNerfSoa& rays_hit = m_render_mode == ERenderMode::Slice ? m_nerf.tracer.rays_init() : m_nerf.tracer.rays_hit();

	if (m_render_mode == ERenderMode::Slice) {
		// Store colors in the normal buffer
		uint32_t n_elements = next_multiple(n_hit, BATCH_SIZE_MULTIPLE);

		m_nerf.vis_input.enlarge(n_elements);
		m_nerf.vis_rgba.enlarge(n_elements);
		linear_kernel(generate_nerf_network_inputs_at_current_position, 0, stream, n_hit, m_aabb, rays_hit.payload.data(), m_nerf.vis_input.data());

		GPUMatrix<float> positions_matrix((float*)m_nerf.vis_input.data(), sizeof(NerfCoordinate)/sizeof(float), n_elements);
		GPUMatrix<float> rgbsigma_matrix((float*)m_nerf.vis_rgba.data(), 4, n_elements);

		if (m_visualized_dimension == -1) {
			m_network->inference(stream, positions_matrix, rgbsigma_matrix);
			linear_kernel(compute_nerf_density, 0, stream, n_hit, m_nerf.vis_rgba.data(), m_nerf.rgb_activation, m_nerf.density_activation);
		} else {
			m_network->visualize_activation(stream, m_visualized_layer, m_visualized_dimension, positions_matrix, rgbsigma_matrix);
		}

		linear_kernel(shade_kernel_nerf, 0, stream,
			n_hit,
			m_nerf.vis_rgba.data(),
			rays_hit.payload.data(),
			m_render_mode,
			m_nerf.training.linear_colors,
			render_buffer.frame_buffer()
		);
		return;
	}

	linear_kernel(shade_kernel_nerf, 0, stream,
		n_hit,
		rays_hit.rgba.data(),
		rays_hit.payload.data(),
		m_render_mode,
		m_nerf.training.linear_colors,
		render_buffer.frame_buffer()
	);

	if (render_mode == ERenderMode::Cost) {
		std::vector<NerfPayload> payloads_final_cpu(n_hit);
		rays_hit.payload.copy_to_host(payloads_final_cpu, n_hit);
		size_t total_n_steps = 0;
		for (uint32_t i = 0; i < n_hit; ++i) {
			total_n_steps += payloads_final_cpu[i].n_steps;
		}
		tlog::info() << "Total steps per hit= " << total_n_steps << "/" << n_hit << " = " << ((float)total_n_steps/(float)n_hit);
	}
}

```