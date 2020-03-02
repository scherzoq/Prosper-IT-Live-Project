# Live Project: Full-Scale Web Application Using ASP.NET MVC and Entity Framework

## Introduction
At the end of my time in The Tech Academy's C# and .NET Framework boot camp, I worked on a two-week live project for Prosper IT Consulting. This involved working with my peers on a team to develop a full-scale web application using ASP.NET MVC and Entity Framework. We worked on an interactive website to manage dynamic content (e.g., productions, cast members, subscribers) for a theater/acting company. The web application served as a content management service (CMS) for users, allowing them to easily manage the content on their website without needing to possess technical expertise.

I worked on stories involving both back-end and front-end functionality (most of the stories I worked on involved both of these) in Visual Studio, using a variety of programming languages: C#, HTML/CSS, JavaScript, SQL. The stories I worked on required that I look for solutions independently, but also coordinate effectively with team members. They taught me the importance of breaking jobs up into smaller tasks while not losing sight of the larger context: specifically, it was critical to understand end goals/functionalities required by the user, and to plan accordingly, but also to be able to focus on the specific tasks that would move the project along. And the project demonstrated to me, again and again, that adaptability – along with the ability to find ways in which to overcome roadblocks – are crucial skills for a software developer to have.

I am proud of my work – and of my growth as a software developer – during this two-week sprint. Below please find more detailed descriptions of the main [stories](#stories) I worked on, along with code snippets and navigation links. Included as well is a [summary of skills](#summary-of-skills) learned/reinforced over the course of the project.

## Stories
* [Admin Settings Function](#admin-settings-function)
* [Admin Settings Reader Helper](#admin-settings-reader-helper)
* [Capture Photo for Cast Member](#capture-photo-for-cast-member)
* [Cast Member Allow Null User](#cast-member-allow-null-user)

### Admin Settings Function
This story involved creating a form on the Admin Dashboard where a user could enter values for various dictionary elements that were part of an Admin Settings JSON file. I also had to create functionality that allowed user input to this form to be written to the JSON file in a useable format. This JSON file contained various values set by the Administrator to help populate the content of the entire site, e.g.:

	{
		"current_season": 4,
		"season_productions": {
			"fall": 12,
			"winter": 8,
			"spring": 9
		},
		"recent_definition": {
			"span": 12,
			"date": "2020-01-28"
		},
		"onstage": 2
	}

My initial code used a front-end solution where I utilized JavaScript/JQuery to convert the form input to a JSON string. I succeeded in getting my code to convert the form input into a JSON string like the one above, but I ran into a roadblock when it came to actually writing the string to the server’s JSON file.

Working to overcome this roadblock was a valuable learning experience for me. I was able to adapt by reworking my code and using a C#-based back-end solution instead. First, I created a model to correspond to the Admin Settings:

	namespace TheatreCMS.Models
	{
		public class AdminSettings
		{
			public int current_season { get; set; }
			public seasonProductions season_productions { get; set; }
			public recentDefinition recent_definition { get; set; }
			public int onstage { get; set; }
			public class seasonProductions
			{
				public int fall { get; set; }
				public int winter { get; set; }
				public int spring { get; set; }
			}
			public class recentDefinition
			{
				public int span { get; set; }
				public DateTime date { get; set; }
			}

		}
	}

I then wrote a method in the Admin controller that: 1. created a new AdminSettings C# object using the dashboard form input, 2. converted this object to a JSON string, and 3. wrote this JSON string to the Admin Settings JSON file (in the same format shown earlier):

	[HttpPost]
        public ActionResult SettingsUpdate(AdminSettings adminSet, seasonProductions seasonProd, recentDefinition recentDef)
        {
            AdminSettings settings = new AdminSettings();
            settings = adminSet;
            settings.season_productions = seasonProd;
            settings.recent_definition = recentDef;

            string newSettings = JsonConvert.SerializeObject(settings, Formatting.Indented);
            newSettings = newSettings.Replace("T00:00:00", "");
            string filepath = Server.MapPath(Url.Content("~/AdminSettings.json"));
            using (StreamWriter writer = new StreamWriter(filepath))
            {
                writer.Write(newSettings);
            }

            return RedirectToAction("Dashboard");
        }

### Admin Settings Reader Helper
For this related story, I created a helper function that – whenever it was called – would retrieve the Admin Settings JSON object from the [Admin Settings Function](#admin-settings-function) story, deserialize it into a C# AdminSettings object (as defined by the model I created), and return that object to whatever code called the function. Here is my code for the helper function:

	public class AdminSettingsReader
	{
		public static AdminSettings CurrentSettings()
		{
			AdminSettings currentSettings = new AdminSettings();     
			string filepath = System.Web.HttpContext.Current.Server.MapPath("~/AdminSettings.json");
			string result = string.Empty;
			using (StreamReader r = new StreamReader(filepath))
			{
				result = r.ReadToEnd();
			}
			currentSettings = JsonConvert.DeserializeObject<AdminSettings>(result);
			return currentSettings;
		}
	}
	
I also implemented this helper function in the Admin Dashboard so that, by default, the input fields on the dashboard would be populated with the current settings. I wrote a brief controller method that uses the helper function to retrieve the current settings –

	public ActionResult Dashboard()
		{
		AdminSettings current = new AdminSettings();
		current = AdminSettingsReader.CurrentSettings();
		return View(current);
		}
				
– and then passes these settings to the view. Here is the relevant code from the CSHTML view file:

	@using (Html.BeginForm("SettingsUpdate", "Admin", FormMethod.Post))
	{
	<div>
	CURRENT SEASON:
	<input type="number" name="Current_Season" value="@Html.DisplayFor(model => model.current_season)"><br />
	SEASON PRODUCTIONS:<br />
	Winter:<input type="number" name="Fall" value="@Html.DisplayFor(model => model.season_productions.fall)"><br>
	Fall:<input type="number" name="Winter" value="@Html.DisplayFor(model => model.season_productions.winter)"><br>
	Spring:<input type="number" name="Spring" value="@Html.DisplayFor(model => model.season_productions.spring)"><br>
	RECENT DEFINITION:<br />
	Span:<input type="number" name="Span" value="@Html.DisplayFor(model => model.recent_definition.span)"><br>
	Date:<input type="date" name="Date" value="@Model.recent_definition.date.ToString("yyyy-MM-dd")"><br>
	ONSTAGE:<input type="number" name="Onstage" value="@Html.DisplayFor(model => model.onstage)"><br />
	<button type="submit">Submit</button>
	</div>
	}

### Capture Photo for Cast Member
For this story, I implemented a helper method for uploading images and converting them to byte array in the Cast Member class to allow a photo to be added (and stored in the database) whenever a cast member was created or edited. I also wrote code so that this photo would be displayed on various pages of the website that were relevant to each cast member.

I worked in conjunction with other team members who were implementing the same helper method to allow production photos to be added in other parts of the website. This allowed for more synchronized, consistent code.

The Edit method for the Cast Member controller required that I add logic to account for whether or not the photo was edited or left unchanged whenever the Edit method was called, and as a result of this, I also needed to make tweaks to other parts of the method:

	[HttpPost]
        [ValidateAntiForgeryToken]
        public ActionResult Edit([Bind(Include = "CastMemberID,Name,YearJoined,MainRole,Bio,Photo,CurrentMember")] CastMember castMember, HttpPostedFileBase file)
        {
            ModelState.Remove("CastMemberPersonID");
            string userId = Request.Form["dbUsers"].ToString();
            if (ModelState.IsValid)
            {
                var currentCastMember = db.CastMembers.Find(castMember.CastMemberID);
                byte[] oldPhoto = currentCastMember.Photo;

                currentCastMember.Name = castMember.Name;
                currentCastMember.YearJoined = castMember.YearJoined;
                currentCastMember.MainRole = castMember.MainRole;
                currentCastMember.Bio = castMember.Bio;
                currentCastMember.CurrentMember = castMember.CurrentMember;

                if (file != null && file.ContentLength > 0)
                {
                    byte[] newPhoto = Helpers.ImageUploader.ImageBytes(file, out string _64);
                    currentCastMember.Photo = newPhoto;
                }
                else
                {
                    currentCastMember.Photo = oldPhoto;
                }
		    castMember.CastMemberPersonID = db.Users.Find(userId).Id;
               
                db.Entry(currentCastMember).State = EntityState.Modified;
                db.SaveChanges();
                return RedirectToAction("Index");
            }
            
            return View(castMember);
        }

In various parts of the website, including the Cast Member index view (CSHTML file), I wrote code that allowed each cast member’s photo to be displayed as a thumbnail. I also implemented an alternative display (“CastMember.jpg”, a default template file) in cases where no photo existed for a cast member:
										
	   @{
		string img = "";
		if (item.Photo != null)
		{
		    byte[] thumbBytes = ImageUploader.ImageThumbnail(item.Photo, 100, 100);
		    var base64 = System.Convert.ToBase64String(thumbBytes);
		    img = String.Format("data:image/png;base64,{0}", base64);
		}
		else
		{
		    img = @Url.Content("~/Content/Images/CastMember.jpg");
		}
	    }
	    <img castImage" src="@img" / style="width:100class=" %">

### Cast Member Allow Null User
In the Cast Member Create/Edit views, a dropdown list was utilized to allow an existing user to be added as a Cast Member. However, the dropdown list did not contain a null option. For this story, I had to revise the code to allow for a null option, since there would not always be a matching user for every Cast Member.

I researched possible solutions and then employed the solution I found to be the most efficient one: a front-end solution that used method overloading (with re: to “DropDownList”) to pass a null-value label (“N/A”) as a parameter.

	<div class="form-group">
                <label class="inputLabel col-md-4"> &nbsp; &nbsp;Username</label>
                <div class="col-md-10 formBox">
                    @Html.DropDownList("dbUsers", (IEnumerable<SelectListItem>)ViewData["dbUsers"], "N/A", htmlAttributes: new { @class = "form-control" })
                </div>
	</div>

I also revised the C# code in the Cast Member controller so that it would appropriately handle the possibility of a null value.

## Summary of Skills
* Researching solutions. I independently researched and implemented solutions for each story I worked on, and gained vital experience in the process of doing so.
* Learning to overcome roadblocks and adaptability. For example, in the [Admin Settings Function](#admin-settings-function) story, I substituted a C# back-end method in place of JavaScript/JQuery to successfuly write to a server-side JSON file.
* Testing/debugging code thoroughly yet efficiently. For example, in the [Capture Photo for Cast Member](#capture-photo-for-cast-member) story, I thoroughly tested the functionality of the Create/Edit pages (creating a cast member with OR without an image; editing an element with an existing image and then changing OR not changing that image) to make sure that the code worked in all cases. I ended up revising and fine-tuning my code to account for the different possibilities.
* Coordinating with team members, especially where consistency was required.
* Workflow planning and version control in the context of a team project.

*Jump to: [Stories](#stories), [Summary of Skills](#summary-of-skills), [Page Top](#the-tech-academy-live-project)*
