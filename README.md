# Ekinox on AWS Marketplace

Sample notebooks, API documentation, and reference payloads for Ekinox's
[AWS Marketplace](https://aws.amazon.com/marketplace) machine learning products.

Everything here is published so you can see exactly how a product behaves
**before** you subscribe: the request format, the response schema, a real
sample input, and the real output it produces.

## Products

| Product | Version | Type | Languages | Regions | Directory |
|---|---|---|---|---|---|
| [Pronunciation Assessment](https://aws.amazon.com/marketplace/pp/prodview-uzfcfzqcmds7i) | 1.0.0 | SageMaker model package | French, Arabic | 16 | [`pronunciation-sagemaker/`](pronunciation-sagemaker/) |

Each directory is self-contained — its `README.md` is the starting point, and
its notebook runs end to end against a subscription to that product.

## Getting started

```bash
git clone https://github.com/EkinoxIO/ekinox-marketplace.git
cd ekinox-marketplace/pronunciation-sagemaker
```

Then open `notebooks/` in Amazon SageMaker Studio, a SageMaker notebook
instance, or any local Jupyter environment with AWS credentials configured.

## License

The sample code and documentation in this repository are licensed under the
[Apache License 2.0](LICENSE).

The products themselves are **not** covered by that license — they are licensed
through their AWS Marketplace listings, under the end user license agreement
shown on each listing page.
