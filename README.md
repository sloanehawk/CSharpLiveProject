# C# Live Project

## Introduction
During a two-week sprint at the tech academy, I worked on a team with other software development students to create an ASP.NET MVC web application using Entity Framework for data access. The web application was designed for a theater company to manage its website without requiring any technical knowledge. The application has multiple areas to manage admin needs, subscriber needs, and general public needs. The site includes information on the current season, past productions, and the current cast members. Below are descriptions of the stories I worked on along with code snippets.

## Stories
* [Create a New Photo Class](#create-a-new-photo-class)
* [Photo Class Display Process](#photo-class-display-process)
* [Admin Settings Dashboard](#admin-settings-dashboard)
* [Sponsors Partial View Cleanup](#sponsors-partial-view-cleanup)

### Create a New Photo Class
I was tasked with creating a specific Photo class that has all the methods and helpers to improve consistency, reduce complexity, and reduce calls to the database. We wanted to implement this new class incrementally and maintain existing classes and methods while the new class was being created. 

First, I created the Photo class model with the following attributes:

    public class Photo
    {
        public int PhotoId { get; set; }
        public byte[] PhotoFile { get; set; }
        public int OriginalHeight { get; set; }
        public int OriginalWidth { get; set; }
        public string Title { get; set; }
    }

Once the new class was created, I added the DbSet to the Identity Models, and created a new controller with scaffolded views based on the Photo class. I added code to the Photo controller for uploading an image:

        //file -> byte[]
        public static byte[] ImageBytes(HttpPostedFileBase file)
        {
            //Convert the file into a System.Drawing.Image type
            Image image = Image.FromStream(file.InputStream, true, true);
            //Convert that image into a Byte Array to facilitate storing the image in a database
            var converter = new ImageConverter();
            byte[] imageBytes = (byte[])converter.ConvertTo(image, typeof(byte[]));
            //return Byte Array
            return imageBytes;
        }

        //byte[] -> smaller byte[]
        public static byte[] ImageThumbnail(byte[] imageBytes, int thumbWidth, int thumbHeight)
        {
            using (MemoryStream ms = new MemoryStream())
            {
                Image img = Image.FromStream(new MemoryStream(imageBytes));
                using (Image thumbnail = img.GetThumbnailImage(img.Width, img.Height, null, new IntPtr()))
                {
                    thumbnail.Save(ms, System.Drawing.Imaging.ImageFormat.Png);
                    return ms.ToArray();
                }
            }
        }

Then I implemented it in the Create method: 

        [HttpPost]
        [ValidateAntiForgeryToken]
        public ActionResult Create([Bind(Include = "PhotoId,PhotoFile,OriginalHeight,OriginalWidth,Title")] Photo photo, HttpPostedFileBase file)
        {
            
            if (ModelState.IsValid)
            {
                byte[] photoArray = ImageBytes(file);
                photo.PhotoFile = photoArray;

                db.Photo.Add(photo);
                db.SaveChanges();
                return RedirectToAction("Index");
            }

            return View(photo);
        }


### Photo Class Display Process 
For this story, my task was to implement the photo display process. This required creating a function in the Photo controller that accepts the id number of the photo to display, then returns an image of that photo to display in the view.

Photo display function:

        public ActionResult DisplayPhoto(int id)
        {
            //find photo object from db
            var photo = db.Photo.Find(id);
            //get byte array 
            var byteData = photo.PhotoFile;
            return File(byteData, "image/png");
        }


Using a Url.Action helper method, I called this function in the index, details, and edit views:

    <img src='@Url.Action("DisplayPhoto", "Photo", new { id = Model.PhotoId })' />




### Admin Settings Dashboard
This story involved modifications to the admin settings dashboard, which initially looked like this:


For each season, the number shown is the ID number of a production. For the ‘onstage’ field, the number represents the ID number of the current production. In order to make this more user friendly, we wanted a dropdown of existing productions from the database, sorted by season number in descending order. 

I added a method in the Admin controller that gets all productions in the database, orders them by season number, and returns a list of SelectListItems. 

        public List<SelectListItem> GetSelectListItems()
        {
            //Create a list of productions sorted by season(int)
            var SortedProductions = db.Productions.ToList().OrderByDescending(prod => prod.Season).ToList();

            // Create an empty list to hold result
            var selectList = new List<SelectListItem>();

            // For each string in the 'elements' variable, create a new SelectListItem object
            // that has both its Value and Text properties set to a particular value.
            foreach (var production in SortedProductions)
            {
                selectList.Add(new SelectListItem
                {
                    Value = production.ProductionId.ToString(),
                    Text = production.Title
                });
            }

            return selectList;
        }


In the Dashboard method, GetSelectListItems() was called and passed to the Dashboard view using ViewData:

        public ActionResult Dashboard()
        {
            #region ViewData
            List<SelectListItem> productionList = GetSelectListItems();
            ViewData["ProductionList"] = productionList;
            #endregion
            AdminSettings current = new AdminSettings();
            current = AdminSettingsReader.CurrentSettings();

            return View(current);
        }

The Dashboard view received the list of SelectListItems, and populated each dropdown using an Html helper: 

    @Html.DropDownListFor(model => model.season_productions.fall, (IEnumerable
            <SelectListItem>)ViewData["ProductionList"])

### Sponsors Partial View Cleanup
Our application inadvertently ended up with two different partial views for the same section, so my task was to consolidate the two methods. 


This required combining the partial views into one partial view:

    @model IEnumerable<TheatreCMS.Models.Sponsor>
    <br />
    <br />

    <h3 class="sponsor-title">Our Sponsors</h3>
    <br />

    @foreach (var item in Model)
    {
        <img class="sponsor-pic" src="data:image;base64,@System.Convert.ToBase64String(item.Logo)" />
        <br />
        <p id="sponsor-text">@Html.DisplayFor(modelItem => item.Name)</p>

    }

    <div class="center">
        <button type="button" class="btn btn-lg sponsor-donate-btn">Donate</button>
    </div>


Then editing the partial view method in the Sponsors controller:

       public ActionResult List()
        {
            return PartialView("_SponsorsPartial", db.Sponsors);
        }

And finally, using an HTML helper to pass the partial view to the Sponsors index.cshtml:

    @Html.ActionLink("View Sponsors", "List")

