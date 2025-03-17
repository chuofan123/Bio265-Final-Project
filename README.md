# Bio265-Final-Project

import os
import math
import numpy as np
import matplotlib.pyplot as plt
from skimage import io, measure, morphology, segmentation, feature
from skimage.filters import threshold_multiotsu
import scipy.ndimage as ndimage
import pandas as pd

########################################
#            Helper Functions          #
########################################

def normalize(im):
    return (im - im.min()) / (im.max() - im.min())

def gamma_correction(im, gamma):
    return normalize(im) ** gamma

def remove_large_nuclei(labels, max_size_factor=2.5):
    """
    For each labeled nucleus, if its area is larger than
    max_size_factor * average_area, remove it (set to 0).
    """
    props = measure.regionprops(labels)
    if not props:
        return labels
    
    areas = [p.area for p in props]
    avg_area = np.mean(areas)
    max_allowed_area = max_size_factor * avg_area

    new_labels = np.zeros_like(labels, dtype=np.int32)
    new_id = 0
    for region in props:
        if region.area <= max_allowed_area:
            new_id += 1
            new_labels[labels == region.label] = new_id
    return new_labels

def segment_binary(im, min_size=10):
    """Labels a binary mask, removes small objects, and clears borders."""
    labeled = measure.label(im)
    labeled = morphology.remove_small_objects(labeled, min_size=min_size)
    labeled = segmentation.clear_border(labeled)
    return measure.label(labeled > 0)

########################################
#    Nucleoli + Nuclei Segmentation    #
########################################

def segment_nucleoli_and_nuclei(
    gfp_image, 
    gamma=1.0, 
    max_size_factor=2.5, 
    nuclei_sensitivity=0.9, 
    nucleoli_sensitivity=0.95
):
    """
    1. Multi-threshold => background, nuclei, nucleoli.
    2. Use watershed to separate touching nuclei, then remove oversized lumps.
    3. Lower the threshold slightly for dim cells, and dilate nuclei to capture them.
    4. Nucleoli are thresholded at a slightly lower boundary for sensitivity.
    5. One nucleolus per nucleus is enforced.
    """
    img_proc = gamma_correction(gfp_image, gamma)
    t1, t2 = threshold_multiotsu(img_proc, classes=3)
    
    nuclei_bin = (img_proc > (t1 * nuclei_sensitivity))
    nuclei_bin = ndimage.binary_fill_holes(nuclei_bin)
    nuclei_bin = morphology.dilation(nuclei_bin, morphology.disk(1))
    
    distance = ndimage.distance_transform_edt(nuclei_bin)
    coords = feature.peak_local_max(distance, labels=nuclei_bin, footprint=np.ones((3,3)))
    mask_local_max = np.zeros_like(distance, dtype=bool)
    mask_local_max[tuple(coords.T)] = True
    markers = measure.label(mask_local_max)
    nuclei_ws = segmentation.watershed(-distance, markers, mask=nuclei_bin)
    nuclei_ws = segmentation.clear_border(nuclei_ws)
    nuclei_ws = measure.label(morphology.remove_small_objects(nuclei_ws, 10) > 0)
    
    nuclei_ws = remove_large_nuclei(nuclei_ws, max_size_factor=max_size_factor)
    
    nucleoli_bin = (img_proc > (t2 * nucleoli_sensitivity))
    nucleoli_mask = segment_binary(nucleoli_bin, min_size=5)
    nucleoli_mask = morphology.dilation(nucleoli_mask, morphology.disk(2))
    nucleoli_mask = measure.label(nucleoli_mask > 0)
    
    final_nuclei = np.zeros_like(nuclei_ws, dtype=np.int32)
    final_nucleoli = np.zeros_like(nucleoli_mask, dtype=np.int32)
    nucleus_id = 0
    for region in measure.regionprops(nuclei_ws, intensity_image=img_proc):
        mask_nucleus = (nuclei_ws == region.label)
        overlap_labels = np.unique(nucleoli_mask[mask_nucleus])
        overlap_labels = overlap_labels[overlap_labels > 0]
        if len(overlap_labels) > 0:
            best_label, best_area = None, 0
            for lab in overlap_labels:
                area = np.sum(nucleoli_mask == lab)
                if area > best_area:
                    best_area = area
                    best_label = lab
            nucleus_id += 1
            final_nuclei[mask_nucleus] = nucleus_id
            final_nucleoli[nucleoli_mask == best_label] = nucleus_id
    return final_nucleoli, final_nuclei

def compute_cell_metrics(gfp_img, nuclei_mask, nucleoli_mask):
    """
    Compute properties for each cell based on final segmentation masks.
    Returns two dictionaries keyed by cell id.
    """
    nuclei_regions = measure.regionprops(nuclei_mask, intensity_image=gfp_img)
    nucleoli_regions = measure.regionprops(nucleoli_mask, intensity_image=gfp_img)
    
    nucleus_metrics = {}
    nucleolus_metrics = {}
    
    for region in nuclei_regions:
        cell_id = region.label
        area = region.area
        mean_intensity = region.mean_intensity
        integrated_intensity = mean_intensity * area
        intensities = [gfp_img[tuple(coord)] for coord in region.coords]
        intensity_std = np.std(intensities)
        cv = intensity_std / mean_intensity if mean_intensity > 0 else 0
        eccentricity = region.eccentricity
        solidity = region.solidity
        perimeter = region.perimeter if region.perimeter > 0 else np.sqrt(area * 4 * np.pi)
        circularity = (4 * np.pi * area) / (perimeter**2) if perimeter > 0 else 0
        centroid = region.centroid
        nucleus_metrics[cell_id] = {'area': area,
                                    'mean_intensity': mean_intensity,
                                    'integrated_intensity': integrated_intensity,
                                    'intensity_std': intensity_std,
                                    'cv': cv,
                                    'eccentricity': eccentricity,
                                    'solidity': solidity,
                                    'circularity': circularity,
                                    'centroid': centroid}
    for region in nucleoli_regions:
        cell_id = region.label
        area = region.area
        mean_intensity = region.mean_intensity
        integrated_intensity = mean_intensity * area
        intensities = [gfp_img[tuple(coord)] for coord in region.coords]
        intensity_std = np.std(intensities)
        cv = intensity_std / mean_intensity if mean_intensity > 0 else 0
        eccentricity = region.eccentricity
        solidity = region.solidity
        perimeter = region.perimeter if region.perimeter > 0 else np.sqrt(area * 4 * np.pi)
        circularity = (4 * np.pi * area) / (perimeter**2) if perimeter > 0 else 0
        centroid = region.centroid
        nucleolus_metrics[cell_id] = {'area': area,
                                      'mean_intensity': mean_intensity,
                                      'integrated_intensity': integrated_intensity,
                                      'intensity_std': intensity_std,
                                      'cv': cv,
                                      'eccentricity': eccentricity,
                                      'solidity': solidity,
                                      'circularity': circularity,
                                      'centroid': centroid}
    return nucleus_metrics, nucleolus_metrics

def get_metrics_dict(gfp_img, nuclei_mask, nucleoli_mask):
    """
    Computes per-cell metrics and aggregates them into lists.
    Returns a dictionary with keys mapping to lists.
    """
    nucleus_metrics, nucleolus_metrics = compute_cell_metrics(gfp_img, nuclei_mask, nucleoli_mask)
    common_ids = sorted(list(set(nucleus_metrics.keys()) & set(nucleolus_metrics.keys())))
    
    metrics = {
        'nucleus_area': [nucleus_metrics[i]['area'] for i in common_ids],
        'nucleolus_area': [nucleolus_metrics[i]['area'] for i in common_ids],
        'nucleus_mean_intensity': [nucleus_metrics[i]['mean_intensity'] for i in common_ids],
        'nucleolus_mean_intensity': [nucleolus_metrics[i]['mean_intensity'] for i in common_ids],
        'nucleus_integrated_intensity': [nucleus_metrics[i]['integrated_intensity'] for i in common_ids],
        'nucleolus_integrated_intensity': [nucleolus_metrics[i]['integrated_intensity'] for i in common_ids],
        'nucleus_cv': [nucleus_metrics[i]['cv'] for i in common_ids],
        'nucleolus_cv': [nucleolus_metrics[i]['cv'] for i in common_ids],
        'nucleus_eccentricity': [nucleus_metrics[i]['eccentricity'] for i in common_ids],
        'nucleolus_eccentricity': [nucleolus_metrics[i]['eccentricity'] for i in common_ids],
        'nucleus_solidity': [nucleus_metrics[i]['solidity'] for i in common_ids],
        'nucleolus_solidity': [nucleolus_metrics[i]['solidity'] for i in common_ids],
        'nucleus_circularity': [nucleus_metrics[i]['circularity'] for i in common_ids],
        'nucleolus_circularity': [nucleolus_metrics[i]['circularity'] for i in common_ids],
        'fraction_area': [nucleolus_metrics[i]['area'] / nucleus_metrics[i]['area'] for i in common_ids],
        'radial_location': [],
        'integrated_ratio': []
    }
    
    for i in common_ids:
        nuc_centroid = np.array(nucleus_metrics[i]['centroid'])
        nucleo_centroid = np.array(nucleolus_metrics[i]['centroid'])
        dist = np.linalg.norm(nuc_centroid - nucleo_centroid)
        radius = math.sqrt(nucleus_metrics[i]['area'] / np.pi)
        norm_dist = dist / radius if radius > 0 else 0
        metrics['radial_location'].append(norm_dist)
        
    for i in common_ids:
        ratio = (nucleolus_metrics[i]['integrated_intensity'] / 
                 nucleus_metrics[i]['integrated_intensity']) if nucleus_metrics[i]['integrated_intensity'] > 0 else 0
        metrics['integrated_ratio'].append(ratio)
    
    return metrics

def plot_combined_metrics(metrics):
    """
    Plots combined metrics across images.
    """
    fig, axes = plt.subplots(4, 3, figsize=(18, 24))
    
    axes[0,0].hist(metrics['fraction_area'], bins=20, color='skyblue')
    axes[0,0].set_title('Fraction of Nuclear Area Occupied by Nucleoli')
    axes[0,0].set_xlabel('Fraction')
    axes[0,0].set_ylabel('Count')
    
    axes[0,1].hist(metrics['nucleus_eccentricity'], bins=20, color='green')
    axes[0,1].set_title('Nucleus Eccentricity')
    axes[0,1].set_xlabel('Eccentricity')
    
    axes[0,2].hist(metrics['nucleus_solidity'], bins=20, color='orange')
    axes[0,2].set_title('Nucleus Solidity')
    axes[0,2].set_xlabel('Solidity')
    
    axes[1,0].hist(metrics['nucleus_circularity'], bins=20, color='purple')
    axes[1,0].set_title('Nucleus Circularity')
    axes[1,0].set_xlabel('Circularity')
    
    axes[1,1].hist(metrics['nucleolus_eccentricity'], bins=20, color='green')
    axes[1,1].set_title('Nucleolus Eccentricity')
    axes[1,1].set_xlabel('Eccentricity')
    
    axes[1,2].hist(metrics['nucleolus_solidity'], bins=20, color='orange')
    axes[1,2].set_title('Nucleolus Solidity')
    axes[1,2].set_xlabel('Solidity')
    
    axes[2,0].hist(metrics['nucleolus_circularity'], bins=20, color='purple')
    axes[2,0].set_title('Nucleolus Circularity')
    axes[2,0].set_xlabel('Circularity')
    
    axes[2,1].hist(metrics['radial_location'], bins=20, color='teal')
    axes[2,1].set_title('Normalized Radial Distance')
    axes[2,1].set_xlabel('Distance (Normalized)')
    
    axes[2,2].hist(metrics['integrated_ratio'], bins=20, color='magenta')
    axes[2,2].set_title('Integrated Intensity Ratio (Nucleolus/Nucleus)')
    axes[2,2].set_xlabel('Intensity Ratio')
    
    axes[3,0].hist(metrics['nucleus_cv'], bins=20, alpha=0.7, label='Nucleus CV', color='blue')
    axes[3,0].hist(metrics['nucleolus_cv'], bins=20, alpha=0.7, label='Nucleolus CV', color='red')
    axes[3,0].set_title('Coefficient of Variation of Intensity')
    axes[3,0].set_xlabel('CV')
    axes[3,0].legend()
    
    axes[3,1].scatter(metrics['nucleus_area'], metrics['nucleolus_area'], color='black', alpha=0.7)
    axes[3,1].set_title('Nucleus Area vs Nucleolus Area')
    axes[3,1].set_xlabel('Nucleus Area')
    axes[3,1].set_ylabel('Nucleolus Area')
    
    fig2, ax2 = plt.subplots(1, 1, figsize=(8, 6))
    ax2.scatter(metrics['nucleus_mean_intensity'], metrics['nucleolus_mean_intensity'], 
                color='darkgreen', alpha=0.7)
    ax2.set_title('Nucleus Mean Intensity vs Nucleolus Mean Intensity')
    ax2.set_xlabel('Nucleus Mean Intensity')
    ax2.set_ylabel('Nucleolus Mean Intensity')
    
    plt.tight_layout()
    plt.show()
    
    # Population-level variability (CV)
    def compute_population_cv(values):
        return np.std(values) / np.mean(values) if np.mean(values) != 0 else 0
    
    metrics_names = ['Area', 'Mean Intensity', 'Integrated Intensity', 'CV', 
                     'Eccentricity', 'Solidity', 'Circularity']
    nucleus_vals = [metrics['nucleus_area'], metrics['nucleus_mean_intensity'],
                    metrics['nucleus_integrated_intensity'], metrics['nucleus_cv'],
                    metrics['nucleus_eccentricity'], metrics['nucleus_solidity'],
                    metrics['nucleus_circularity']]
    nucleolus_vals = [metrics['nucleolus_area'], metrics['nucleolus_mean_intensity'],
                      metrics['nucleolus_integrated_intensity'], metrics['nucleolus_cv'],
                      metrics['nucleolus_eccentricity'], metrics['nucleolus_solidity'],
                      metrics['nucleolus_circularity']]
    
    nucleus_cv_pop = [compute_population_cv(v) for v in nucleus_vals]
    nucleolus_cv_pop = [compute_population_cv(v) for v in nucleolus_vals]
    
    fig3, ax3 = plt.subplots(1, 1, figsize=(10, 6))
    index = np.arange(len(metrics_names))
    bar_width = 0.35
    ax3.bar(index, nucleus_cv_pop, bar_width, label='Nucleus')
    ax3.bar(index + bar_width, nucleolus_cv_pop, bar_width, label='Nucleolus')
    ax3.set_xlabel('Metric')
    ax3.set_ylabel('Coefficient of Variation')
    ax3.set_title('Population-level Variability of Metrics')
    ax3.set_xticks(index + bar_width / 2)
    ax3.set_xticklabels(metrics_names)
    ax3.legend()
    
    plt.tight_layout()
    plt.show()

def load_images(gfp_image_path, nuclei_mask_path, nucleoli_mask_path):
    """
    Loads the GFP image and corresponding segmentation masks using relative paths.
    """
    if not os.path.exists(gfp_image_path):
        raise FileNotFoundError(f"GFP image not found: {gfp_image_path}")
    if not os.path.exists(nuclei_mask_path):
        raise FileNotFoundError(f"Nuclei mask not found: {nuclei_mask_path}")
    if not os.path.exists(nucleoli_mask_path):
        raise FileNotFoundError(f"Nucleoli mask not found: {nucleoli_mask_path}")
    
    gfp_img = io.imread(gfp_image_path)
    nuclei_mask = np.load(nuclei_mask_path)
    nucleoli_mask = np.load(nucleoli_mask_path)
    return gfp_img, nuclei_mask, nucleoli_mask

def analyze_yeast_images(output_dir=None):
    """
    Processes images from GFP_02.tif to GFP_11.tif and returns a results dictionary.
    """
    results = {
        'image_pairs': [],
        'nucleoli_count': [],
        'nuclei_count': [],
        'nucleoli_per_nucleus': [],
        'nucleoli_sizes': [],
        'nuclei_sizes': [],
        'nucleoli_intensities': [],
        'nuclei_intensities': []
    }
    
    for i in range(2, 12):
        gfp_file = f"GFP_{i:02d}.tif"
        bf_file = f"BF_{i:02d}.tif"
        print(f"Processing {gfp_file} and {bf_file}")
        gfp_img = io.imread(gfp_file)
        bf_img = io.imread(bf_file)
        
        nucleoli_mask, nuclei_mask = segment_nucleoli_and_nuclei(
            gfp_img, 
            gamma=1.0, 
            max_size_factor=2.5,
            nuclei_sensitivity=0.9,
            nucleoli_sensitivity=0.95
        )
        
        nucleoli_count = np.max(nucleoli_mask)
        nuclei_count = np.max(nuclei_mask)
        nucleoli_props = measure.regionprops(nucleoli_mask, intensity_image=gfp_img)
        nuclei_props = measure.regionprops(nuclei_mask, intensity_image=gfp_img)
        
        nucleoli_sizes = [p.area for p in nucleoli_props]
        nuclei_sizes = [p.area for p in nuclei_props]
        nucleoli_intensities = [p.mean_intensity for p in nucleoli_props]
        nuclei_intensities = [p.mean_intensity for p in nuclei_props]
        
        nucleoli_per_nucleus = []
        for nid in range(1, nuclei_count + 1):
            mask_nucleus = (nuclei_mask == nid)
            these_nucleoli = np.unique(nucleoli_mask[mask_nucleus])
            these_nucleoli = these_nucleoli[these_nucleoli > 0]
            nucleoli_per_nucleus.append(len(these_nucleoli))
        
        results['image_pairs'].append((gfp_file, bf_file))
        results['nucleoli_count'].append(nucleoli_count)
        results['nuclei_count'].append(nuclei_count)
        results['nucleoli_per_nucleus'].append(nucleoli_per_nucleus)
        results['nucleoli_sizes'].append(nucleoli_sizes)
        results['nuclei_sizes'].append(nuclei_sizes)
        results['nucleoli_intensities'].append(nucleoli_intensities)
        results['nuclei_intensities'].append(nuclei_intensities)
        
        if output_dir:
            fig, axes = plt.subplots(2, 3, figsize=(18, 12))
            axes[0,0].imshow(gfp_img, cmap='magma')
            axes[0,0].set_title('GFP Channel')
            axes[0,1].imshow(bf_img, cmap='cividis')
            axes[0,1].set_title('Brightfield Channel')
            
            overlay = np.zeros((*gfp_img.shape, 3), dtype=np.uint8)
            overlay[...,0] = (normalize(gfp_img)*255).astype(np.uint8)
            for nl_id in range(1, nucleoli_count + 1):
                overlay[nucleoli_mask == nl_id, 2] = 255
            for nc_id in range(1, nuclei_count + 1):
                mask_nuc = (nuclei_mask == nc_id)
                boundary = morphology.binary_dilation(mask_nuc) & ~mask_nuc
                overlay[boundary,1] = 255
            axes[0,2].imshow(overlay)
            axes[0,2].set_title('Segmentation Overlay')
            
            axes[1,0].imshow(nucleoli_mask > 0, cmap='Blues')
            axes[1,0].set_title('Nucleoli Mask')
            axes[1,1].imshow(nuclei_mask > 0, cmap='Greens')
            axes[1,1].set_title('Nuclei Mask')
            
            axes[1,2].hist(nucleoli_intensities, bins=20, alpha=0.5, label='Nucleoli')
            axes[1,2].hist(nuclei_intensities, bins=20, alpha=0.5, label='Nuclei')
            axes[1,2].set_title('Intensity Distribution')
            axes[1,2].set_xlabel('Mean Intensity')
            axes[1,2].set_ylabel('Count')
            axes[1,2].legend()
            
            plt.tight_layout()
            plt.savefig(os.path.join(output_dir, f'segmentation_result_{i:02d}.png'), dpi=150)
            plt.close()
            
            np.save(os.path.join(output_dir, f'nucleoli_mask_{i}.npy'), nucleoli_mask)
            np.save(os.path.join(output_dir, f'nuclei_mask_{i}.npy'), nuclei_mask)
    
    return results

def compute_summary(results):
    """
    Computes combined summary statistics across all images.
    """
    all_nucleoli_sizes = [s for sizes in results['nucleoli_sizes'] for s in sizes]
    all_nuclei_sizes = [s for sizes in results['nuclei_sizes'] for s in sizes]
    all_nucleoli_intensities = [i for intensities in results['nucleoli_intensities'] for i in intensities]
    all_nuclei_intensities = [i for intensities in results['nuclei_intensities'] for i in intensities]
    all_nucleoli_per_nucleus = [count for counts in results['nucleoli_per_nucleus'] for count in counts]
    
    summary = {
        "Total nucleoli (count)": sum(results["nucleoli_count"]),
        "Total nuclei (count)": sum(results["nuclei_count"]),
        "Average nucleoli per nucleus": np.mean(all_nucleoli_per_nucleus) if all_nucleoli_per_nucleus else 0,
        "Average nucleolus size": np.mean(all_nucleoli_sizes) if all_nucleoli_sizes else 0,
        "Average nucleus size": np.mean(all_nuclei_sizes) if all_nuclei_sizes else 0,
        "Average nucleolus intensity": np.mean(all_nucleoli_intensities) if all_nucleoli_intensities else 0,
        "Average nucleus intensity": np.mean(all_nuclei_intensities) if all_nuclei_intensities else 0
    }
    return summary

########################################
#                Main                  #
########################################

if __name__ == "__main__":
    output_dir = "segmentation_results"
    results = analyze_yeast_images(output_dir=output_dir)
    
    # Compute combined summary statistics
    summary = compute_summary(results)
    print("Combined Summary Statistics:")
    for key, value in summary.items():
        print(f"{key}: {value:.2f}")
    
    # Save combined summary to CSV
    df_summary = pd.DataFrame([summary])
    output_csv = "combined_summary_statistics.csv"
    df_summary.to_csv(output_csv, index=False)
    print(f"Combined summary statistics saved to {output_csv}")
    
    # Optionally, combine per-cell metrics across images and plot them
    all_metrics_list = []
    for i in range(2, 12):
        gfp_file = f"GFP_{i:02d}.tif"
        nuclei_mask_path = os.path.join("segmentation_results", f"nuclei_mask_{i}.npy")
        nucleoli_mask_path = os.path.join("segmentation_results", f"nucleoli_mask_{i}.npy")
        gfp_img = io.imread(gfp_file)
        nuclei_mask = np.load(nuclei_mask_path)
        nucleoli_mask = np.load(nucleoli_mask_path)
        # Use the per-cell metrics function
        from copy import deepcopy
        metrics = compute_cell_metrics(gfp_img, nuclei_mask, nucleoli_mask)[0]  # using nuclei metrics as example
        # Here, we simply collect the metrics dictionary for each image (you can modify as needed)
        all_metrics_list.append(deepcopy(metrics))
    
    # (Additional combined plotting can be added here if desired.)
