# Architecture Diagram

> Requires Mermaid v10+ for `namespace` support (GitHub, GitLab, Obsidian, most modern renderers).

```mermaid
classDiagram
  namespace iddata {
    class DiseaseDataLoader {
      +load(sources, as_of, ancillary, drop_pandemic_seasons) DataFrame
    }
    class DataSource {
      <<abstract>>
      +load(as_of) DataFrame
      +source_name() SourceType
    }
    class NHSNDataSource {
      +load(as_of) DataFrame
    }
    class NSSPDataSource {
      +load(as_of) DataFrame
    }
    class FluSurvNetDataSource {
      +load(as_of) DataFrame
    }
    class ILINetDataSource {
      +load(as_of) DataFrame
    }
    class AncillaryData {
      <<abstract>>
      +load() DataFrame
    }
    class PopulationData {
      +load() DataFrame
    }
  }

  DataSource <|-- NHSNDataSource
  DataSource <|-- NSSPDataSource
  DataSource <|-- FluSurvNetDataSource
  DataSource <|-- ILINetDataSource
  AncillaryData <|-- PopulationData
  DiseaseDataLoader ..> DataSource : uses
  DiseaseDataLoader ..> AncillaryData : uses

  namespace idmodels {
    class IDModel {
      <<abstract>>
      +run(run_config) None
      #_build_sources(run_config) list
      #_build_feature_pipeline(run_config) FeaturePipeline
      #_fit_and_predict(df, feat_names, run_config) DataFrame
    }
    class SARIXModel {
      +run(run_config) None
    }
    class SARIXFourierModel
    class GBQRModel {
      +run(run_config) None
    }
    class Transform {
      <<abstract>>
      +apply(df) DataFrame
      +invert(values, context) ndarray
    }
    class FourthRootTransform {
      +apply(df) DataFrame
      +invert(values, context) ndarray
    }
    class IdentityTransform {
      +apply(df) DataFrame
      +invert(values, context) ndarray
    }
    class SourceScaleTransform {
      +apply(df) DataFrame
      +invert(values, context) ndarray
    }
    class CenterScaleTransform {
      +apply(df) DataFrame
      +invert(values, context) ndarray
    }
    class ComposedTransform {
      +apply(df) DataFrame
      +invert(values, context) ndarray
    }
    class Feature {
      <<abstract>>
      +apply(df, feat_names) tuple
    }
    class LagFeature {
      +apply(df, feat_names) tuple
    }
    class RollingMeanFeature {
      +apply(df, feat_names) tuple
    }
    class TaylorFeature {
      +apply(df, feat_names) tuple
    }
    class HorizonTargetFeature {
      +apply(df, feat_names) tuple
    }
    class OneHotEncodingFeature {
      +apply(df, feat_names) tuple
    }
    class HolidayFeature {
      +apply(df, feat_names) tuple
    }
    class LevelFeatureFilter {
      +apply(df, feat_names) tuple
    }
    class DirectionalWaveFeature {
      +apply(df, feat_names) tuple
    }
    class FeaturePipeline {
      +apply(df) tuple
    }
    class ModelConfig {
      <<abstract>>
      +model_name str
      +sources list
      +power_transform PowerTransform
    }
    class SARIXModelConfig {
      +p int
      +P int
      +d int
      +D int
      +season_period int
      +theta_pooling PoolingStrategy
      +sigma_pooling PoolingStrategy
      +num_warmup int
      +num_samples int
    }
    class SARIXFourierModelConfig {
      +fourier_K int
      +fourier_pooling PoolingStrategy
    }
    class GBQRModelConfig {
      +num_bags int
      +bag_frac_samples float
      +incl_level_feats bool
      +use_directional_waves bool
    }
    class RunConfig {
      +disease Disease
      +ref_date date
      +output_root Path
      +max_horizon int
      +states list
      +q_levels list
    }
  }

  IDModel <|-- SARIXModel
  IDModel <|-- GBQRModel
  SARIXModel <|-- SARIXFourierModel
  Transform <|-- FourthRootTransform
  Transform <|-- IdentityTransform
  Transform <|-- SourceScaleTransform
  Transform <|-- CenterScaleTransform
  Transform <|-- ComposedTransform
  ComposedTransform o-- Transform : contains
  Feature <|-- LagFeature
  Feature <|-- RollingMeanFeature
  Feature <|-- TaylorFeature
  Feature <|-- HorizonTargetFeature
  Feature <|-- OneHotEncodingFeature
  Feature <|-- HolidayFeature
  Feature <|-- LevelFeatureFilter
  Feature <|-- DirectionalWaveFeature
  FeaturePipeline o-- Feature : contains
  ModelConfig <|-- SARIXModelConfig
  ModelConfig <|-- GBQRModelConfig
  SARIXModelConfig <|-- SARIXFourierModelConfig
  IDModel ..> DiseaseDataLoader : uses
  IDModel ..> FeaturePipeline : uses
  IDModel ..> Transform : uses
  IDModel ..> RunConfig : param
  SARIXModel ..> SARIXModelConfig : configured by
  SARIXFourierModel ..> SARIXFourierModelConfig : configured by
  GBQRModel ..> GBQRModelConfig : configured by

  namespace operational-models {
    class covid_ar6_pooled {
      +main(today_date, short_run)
      Rscript 1_plot.R
    }
    class covid_gbqr {
      +main(today_date, short_run)
      Rscript 1_plot.R
    }
    class flu_ar2 {
      +main(today_date, short_run)
      Rscript plot.R
    }
    class flu_flusion {
      +main(today_date, short_run)
      Rscript 2_flusion_ensemble.R
      Rscript 3_plot.R
    }
    class flu_trends_ensemble {
      +main(today_date)
      Rscript main.R
    }
  }

  covid_ar6_pooled ..> SARIXModel : AR(6) pooled
  covid_gbqr ..> GBQRModel : instantiates
  flu_ar2 ..> SARIXModel : AR(2) per-location
  flu_flusion ..> SARIXModel : AR(6) pooled
  flu_flusion ..> GBQRModel : ensemble
  flu_trends_ensemble ..> RunConfig : uses
```
