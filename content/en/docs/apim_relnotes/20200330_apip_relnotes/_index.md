---
title: API Portal 7.7 March 2020 Release Notes
linkTitle: API Portal 7.7 March 2020
no_list: true
weight: 80
date: 2020-03-11T00:00:00.000Z
description: Learn about the new features and enhancements in this update of API Portal.
---

## Summary

API Portal provides an API consumer-facing interface that you can customize to match your corporate brand. API Portal is a layered product linked to API Manager, and requires both API Manager and API Gateway. For more information, see the API Gateway and API Manager documentation.

## Installation

API Portal is available as a software installation or a virtualized deployment in a Docker container. For more information, see the following options:

* If you are installing API Portal for the first time using this update, see [Install API Portal](https://axway-open-docs.netlify.com/docs/apim_installation/apiportal_install/)
* If you are already using API Portal (7.5.x, 7.6.x, 7.7.x) and want to install this update, see [Upgrade API Portal](/docs/apim_installation/apiportal_install/upgrade_automatic/)
* If you want to deploy API Portal in Docker containers, see [Deploy API Portal in containers](/docs/apim_installation/apiportal_docker/)

## New features and enhancements

The following new features and enhancements are available in this update.

### API Details page improvements

* You can set a payload limit size to download the response as a file if the response exceeds the size you have set.
* You can configure the colors of the different methods for both SOAP and REST APIs.
* You can set a flag to include or exclude the value for the `externalDocs` attribute. The flag defaults to **On**. When the flag is enabled, the value for the externalDocs attribute is appended to the **Description** field and rendered as part of that field.
* You can configure whether to show **Try it** for groups of HTTP methods per connected API Manager instance. This allows for finer grained control of the **Try it** functionality.

For more information, see [Customize API Catalog](https://axway-open-docs.netlify.com/docs/apim_administration/apiportal_admin/customize_apicatalog_overview/)

## Limitations of this update

This update has the following limitations:

* API Portal 7.7.20200330 is compatible with API Gateway and API Manager 7.7.20200330 only.
* Upgrade to API Portal 7.7.20200330 is supported from API Portal 7.7.0 only. To upgrade from earlier versions (for example, 7.5.x, 7.6.x) you must first upgrade to API Portal 7.7.
* The ready-made API Portal Docker image 7.7.20200330 is strictly for development environments only, and it is not recommended for use in production environments.

    * It is not recommended to use the image in production environments because the image is built with CentOS as a base OS, and our Axway security scans have detected multiple security concerns with this OS. We continue to monitor the latest versions of this base OS to determine if these issues have been resolved, but until we can ship a hardened image that passes our security concerns, we cannot advise customers to use this image in a production environment. Delivery a Docker image for production use is already planned in the [API Portal 2020 roadmap](https://community.axway.com/s/api-portal).
* Upgrading from previous API Portal Docker image is not supported.
* This update is not available as a virtual appliance or as a managed service on Axway Cloud.

## Removed features

* Documentation is no longer provided in PDF format. You can continue to save individual topics or entire guides in PDF format using the **Save as PDF** icon on the [Axway documentation portal](https://docs.axway.com/).

## Fixed issues

See [Fixed issues](/docs/apim_relnotes/apip_cumulativeissues/apip_relnotes_cumulativeissues/) for a complete list.

## Known issues

The following are known issues in this version of API Portal.

### Page layout and alignment for Arabic language

Changing the API Portal language to Arabic (or any other right to left language) results in issues with page layout and alignment on the API Portal Home and Pricing pages, and some buttons are not visible. As a workaround, you can turn on the development mode in JAI. Follow these steps:

1. Log in to Joomla! Admin Interface (JAI).
2. In the JAI top navigation bar, click **Extensions > Templates**.
3. Click your template style (for example, `purity_III * Default`) to open it.
4. Click the **General** tab.
5. Change **Development Mode** to `ON`.
6. Click **Save** and click **Close** to close the template style.

Related Issue: IAP-308

### Sign in button is shown after user logs in to API Portal

After user logs in to API Portal, the **Sign in** button is shown in the banner section instead of the **Explore** button.

Related Issue: IAP-3037

## Documentation

This section describes documentation enhancements and related documentation.

### Documentation enhancements

The latest version of API Gateway, API Manager, and API Portal documentation has been migrated to Markdown format and is available in a [public GitHub repository](https://github.com/Axway/axway-open-docs) to prepare for future collaboration using an open source model. As part of this migration, the documentation has been restructured to help users navigate the content and find the information they are looking for more easily.

Documentation change history is now stored in GitHub. To see details of changes on any page, click the link in the **Last modified** section at the bottom of the page.

### Related documentation

To find all available documents for this product version:

1. Go to <https://docs.axway.com/bundle>.
2. In the left pane Filters list, select your product or product version.

Customers with active support contracts need to log in to access restricted content.

The following reference documents are also available:

* [Supported Platforms](https://docs.axway.com/bundle/Axway_Products_SupportedPlatforms_allOS_en) - Lists the different operating systems, databases, browsers, and thick client platforms supported by each Axway product.
* [Interoperability Matrix](https://docs.axway.com/bundle/Axway_Products_InteroperabilityMatrix_allOS_en) - Provides product version and interoperability information for Axway products.

## Support services

The Axway Global Support team provides worldwide 24 x 7 support for customers with active support agreements.

Email [support@axway.com](mailto:support@axway.com) or visit [Axway Support](https://support.axway.com/).

See [Get help with API Gateway](/docs/apim_administration/apigtw_admin/trblshoot_get_help/) for the information that you should be prepared to provide when you contact Axway Support.
