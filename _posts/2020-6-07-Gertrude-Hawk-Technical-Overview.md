---
layout: post
title: Gertrude Hawk Technical Overview
---


# Gertrude Hawk Technical Overview
As a Magento technical lead, you always want to work on complex issues, build big things that have a lasting impact, and help your clients reach their end goals.

When I was handed Gertrude Hawk Chocolates in the Spring of 2019, I was excited but a little trepidatious about the complexity of the Fundraising portion of the project. Moving a website off of a Cold Fusion integration and to Magento 2 is complicated in itself. Making a fundraising component with this broad array of features was genuinely going to be one of the most challenging builds I had ever done.

The team and I at Weidenhammer knew that we needed to cover the bases and build something which would stand a test of time but be flexible enough to support new features.

I am not going to dive too deeply into the code behind a lot of this functionality, as it is proprietary to Gertrude Hawk. I will cover from a high-level how to approach a problem with this level of complexity.

## The Problem
Gertrude Hawk had a fundraising system that worked, in a way, on Cold Fusion but was done mostly by hand. Their team had a lot of manual work done within their offices and the warehouses making all of this work. Over ten million dollars worth of business a year, and we wanted to find a way to turn that process digital, as automated as possible, and flexible. That led me to a scenario I wrote down on paper as follows.


Manage 13 million dollars of school fundraising through Magento 2. 

* Parents need to be able to see the sales of their children
* Chair people from the schools need to be able to see sales
* Administrators from Gertrude Hawk need to be able to manage pricing, available catalogs and free shipping on multiple vectors
* It needs to be responsive not just as a mobile platform, but it needs to be fast
* It has to be designed so that users of all technical levels can intuitively understand how it works
* Students/Children need to have a way to attribute their sales to the specific Organization

How do you make custom catalogs in Magento 2, with custom pricing for each Catalog, where we can track who the sale would be attributed to? While I cannot say I built this in the best way that could have ever been done, I met the criteria, and we improved the client's mobile transactions by 2400%, and no, that is not a typo.

## The Approach

I was told many years ago by another engineer that good software starts with slow, measured, meticulous planning. I took that same approach here. I used yellow legal pads to scribble out ideas of how this might work, how the data structure would work, and how it might be built. I knew I would not be able to create all of this as I was running several projects, but I would build some it, scope it all and assign it.

I needed to find a way to get my team in a place to execute on my plan while working with developers who are less skilled than me, as well as lacking all the knowledge from the discovery phase I was involved with. Writing proper Acceptance Criteria will only get you so far; in the end, it comes down to being an effective communicator of your requirements.

I developed this strategy:
1. Prioritize and scope
2. Get someone executing what is scoped
3. Frequent touch points with the developers working the code
4. Check their nightly pushes to make sure it is not going off course

I had the advantage of a great Solutions Architect and a leadership team who understood the concept of decentralized command. I knew the requirements; I knew what needed to be done, so they trusted me to get the project built and delivered. I have always found that being the one demonstrating what was built to the client drives me to get it done right.

### User Stories - A sample
Below are the user stories just for the organization piece:

1. As a site administrator, I need organizations to exist so that I can control which catalogs the organizations have access to

We only had one user story from discovery, a takeaway from this during retrospective is that if something this complex has one user story, it might be underestimated.

### Acceptance Criteria - A sample
Below are the acceptance criteria for the organization piece of work.

1. Hammer_Fundraising Module exists

2. Field in Admin -> Store -> Config -> Weidenhammer -> Fundraising -> Enabled exists
   1. Disabling this removes all functionality

3. hammer_organization_table exists
   1. This table collects the following data:
      1. row_id 
      2. Organization Enabled
      3. Organization Name
      4. Organization Address
      5. Organization Catalog Assigned
      6. Organization Notes
      7. Chair Person (linked to customer account)
      8. Gertrude Hawk Organization ID (6 characters) - Should have UNIQUE flag
4. Customer Attribute exists and is visible to show if Customer is the Chair Person of an Organization
   1. This should only be visible on the backend
5. Order Attribute Exists to show Order belongs to Organization
6. During checkout, the quote is linked to the unique six-digit org code
   1. This is difficult to verify so test this by placing an order

I wrote this as all Acceptance Criteria should be written, as though they have been completed already. I was also dedicated to writing this as in a client agnostic way if it needed to be refactored for use late on.

### Data storage
This is where it began. How to store the data.

I knew I wanted the data to be stored in a series of linked tables. I do not like to over complicate things, but I initially wanted to take advantage of Magento's extension attributes (more on this why this did not work later). So I broke this down into it's smallest parts.

1. Fundraising Organizations needed to Exist
2. Organizations needed to have catalogs of products to sell from, with custom pricing
3. Each Catalog needed to store the products assigned to it, and their custom prices
4. Parents would need a way to attach their kids to Organizations and track their sales, so they get credit.

Four tables and I created an `Extension Attribute` table as well to store the linking data I would use.

### Organizations
For Gertrude Hawk, the Organizations were any school that was doing a fundraiser. They would be created in the back end, a chairperson assigned, and then they needed to have a store front for each student (seller). These all needed a unique six-digit code so that this data could be exported to their internal ERP integration. Each Organization could only ever have one Catalog at a time.

### Catalogs
There needed to be an unlimited number of catalogs. These catalogs needed to have the ability to select products, assign custom pricing, and could be assigned multiple times to organizations. For the display of the Catalog, we took advantage of the existing Product Listing Pages layout that Magento already has. Of course, I overrode the price that was done with an around plugin. 

```php
/**
     * Get custom price
     * @param \Magento\Catalog\Model\Product $subject
     * @param callable $proceed
     * @return callable|string
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function aroundGetPrice(\Magento\Catalog\Model\Product $subject, callable $proceed)
    {
        if (!$this->helper->isModuleEnabled()) {
            return $proceed();
        }

        //check if fundraising store && if organization id is set
        $fundraisingStore = $this->organizationHelper->isFundraisingStore();
        $orgId = $this->organizationHelper->getCustomerOrganizationId();

        if ($fundraisingStore === false || $orgId === null) {
            return $proceed();
        }

        $productId = $subject->getId();
        $catalogId = $this->organizationHelper->getOrganizationCatalogId($orgId);
        $fundraisingPrice = $this->fundraisingCatalogHelper->getFundraisingCatalogProductPrice($catalogId, $productId) ?? $proceed();

        return $fundraisingPrice;
    }
```

Notice this code checks to see if the module is enabled; this is the first mark of a reasonable extension, in my opinion.

### Sellers
Each parent needed to be able to assign their children to a specific school, then they needed a "storefront" of some sort for people to buy products, and that child gets the credit. This needed to be simple enough that any parent could work it, quickly on top of everything else they have to do.

I decided to use Fastly to my advantage with sellers. I would output the Organization's six-digit code and the seller's ID from their entity table to the URL, thus making the seller store a cached page. 

## The pain points
Invariably even the best-architected solution will run into trouble. Mine started the admin; though they certainly did not end there. 

### The Admin
I wanted to take liberal advantage of the Admin UI components that are used in a myriad of places. The first struggle I hit almost instantly was that without doing customization, you could not `re-use` a column. Sadly, a much more significant hurdle came along later. I wanted to display a UX for adding products to the Catalog. The UI components in the admin are suitable for basic CRUD operations but lack the depth of a real UX. So I defaulted (for time) to using a much older paradigm used in Magento 1 known as the Block view.

Inside of the main Catalog view, I used a UI component within a UI component to list all the products assigned to that Catalog, with the controls to remove them present as well as edit their custom price. 

![UI Component Inception]({{ site.baseurl }}/images/assets/Fundraising_Catalog_Exmple.png)

### Extension Attributes
I learned early on in Magento 2 that the service layer is an idea, not a rule. Extension Attributes are a great tool, but you cannot rely on them always being present. For example, when you first land on the checkout page of a Magento 2 site, it quickly runs a `getTotals` and `getShipping` set of instructions to calculate initial shipping rates. These will not include extension attributes. When you click next or fill out the fields, then the Extension Attributes will be present. Sadly, before the final review stage of the checkout, it does another shipping check, and the extension attributes are again absent. While the developer could do some customization to include them in those checks, it is sometimes more efficient (budget-wise) to simply create a different method that runs on observers or plugins. I decided my team would go with an observer for much of the front end checkout experience, even though I wanted to go with extension attributes early on.

We observed `sales_model_service_quote_submit_before` to set the seller & organizational data onto the Order.

```php
/**
     * @param Observer $observer
     * @observes sales_model_service_quote_submit_before
     * @return void
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function execute(Observer $observer)
    {
        if ($this->organizationHelper->isFundraisingStore()) {
            $order = $observer->getEvent()->getOrder();
            /** @var $order \Magento\Sales\Model\Order * */

            $quote = $observer->getEvent()->getQuote();
            /** @var $quote \Magento\Quote\Model\Quote * */

            $sellerId = $quote->getData(OrderSeller::SELLER_FIELD_NAME);

            //in the event a seller is not stored on the quote lets pull off the session
            if ($sellerId === null) {
                $sellerId = $this->organizationHelper->getCustomerSellerId();
            }

            //load the seller entity
            $seller = $this->mySellersRepository->getById($sellerId);
            //get the seller name
            $sellerName = $seller->getSellerName() . ' ' . $seller->getSellerLastname();

            $organizationId = $quote->getData(Organization::ORGANIZATION_ID);


            //if orgId is not present pull it from the session
            if ($organizationId === null) {
                $organizationId = $this->organizationHelper->getCustomerOrganizationId();
            }

            $organizationName = $this->organizationHelper->getOrganizationName($organizationId);

            $order->setData(OrderSeller::SELLER_FIELD_NAME, $sellerId);
            $order->setData(OrderSeller::SELLER_NAME_FIELD, $sellerName);
            $order->setData(Organization::ORGANIZATION_ID, $organizationId);
            $order->setData(Organization::ORGANIZATION_NAME, $organizationName);
        }
    }
```

## Rome was not built in a day
As I mentioned earlier, I am not doing a full technical deep dive, as much of this code is proprietary. This does give some idea of the complexity required to accomplish this work. 

It took three months and two hundred working hours to deliver this. It was mission-critical and an MVP for their build. It is essential during any large project to continually keep your eye on the ball; Not just the budget, burn rate, and end date. You need to make sure that your developers are pushing for an MVP (minimum viable product) to deliver something to the client they can use. Typically I say the real MVP is a website that has products and takes money, but it usually more complicated than that. This is definitely a case where we were doing mostly two sites in one, a large multi-million dollar retail storefront and the fundraising piece, which was a site unto itself. 

When scoping work for other developers, it is sometimes hard to transfer everything you know into a ticket or card for them. I find that means I did not make it small enough in scope. A smaller number of Acceptance Criteria, the more modest the scope, and the easier to explain. However, doing it this way means that someone needs not just to monitor the code but also be in charge of stitching this big monster together at the end to make sure everything play's nice together. 

## Code Review
When I work as a technical lead, I am `the gatekeeper,` which means I am the only person who deploys to production and staging. I look at all code before it gets merged, and I look for items at three levels.

1. Must Fix
   1. This is usually a logic failure or a security vulnerability, or this code breaks a Magento coding standard
      1. These are always handed back to be fixed and often involves more than a comment they typically require a phone call or meeting
2. Should Fix
   1. This is code that does not align with my coding standards, it likely works but is not as efficient as it should be
   2. Missing docblocks fall in this category
   3. Should fix items are usually handed back to be fixed
3. Can Fix
   1. These are purely cosmetic items. Like variable length or using `isset` instead of a null coalesce operator (??)
   2. These are minor things that annoy me as an engineer that I have developers fix if they have other stuff to fix

When it comes to testing, I do not test the developer's code. I do all code reviews locally in PHP Storm, but I do not check it. If it passes code review, I will slate it for deployment. Once the code has been deployed, I assign it back to the developer to configure and test on staging, then tell me it is done. Depending on the developer's experience level, I will then check each AC and make sure it passes before moving it to User Acceptance or Quality Assurance.

This breeds the mindset of testing your code. It forces the developer to review the AC against what it does on staging. I cannot tell you the number of times I have heard, "it worked for me locally," and I am confident it did. Local, no matter how close (especially for Magento Cloud projects), is never going to be the same as being on live hardware. All kinds of variables come in to play once it is deployed; that is why I force the developer to test it themself. 

## The outcome
Gertrude Hawk has been quite successful this year. They have seen their mobile traffic increase by 2400%; they have seen their customer satisfaction rates from the website go up, they have seen more fundraising traffic than ever before. Is this all due to the module I designed and help implement? I doubt it, much of it will be down to marketing, but I like to think some of what was built helps drive that satisfaction. I have personally bought chocolate from them on a Thursday and had it arrive on the next Friday with an ice pack still cold. They are on target for a great fundraising year, even in light of COVID-19, so this build was a complete success.

I hope this has given you the reader some insight into how to tackle a complex problem as well as how to execute that plan with your team.